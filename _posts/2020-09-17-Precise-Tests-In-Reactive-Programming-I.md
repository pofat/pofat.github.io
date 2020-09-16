---
layout: post
title: '如何更精準地測試 Reactive Programming（一）'
date: 2020-09-17
author: Pofat
color: rgb(214,12,142)
cover: 'https://i.imgur.com/pkK3Ekw.jpg'
tags: RxSwift Swift
---



Functional Reactive Programming 顯然是這些年的開發模式顯學之一，尤其是 Apple 也推出了自己的方案 - [Combine](https://developer.apple.com/documentation/combine)，由此可見其重要性。基本上 Reactive Programming 是將非同步的事件，加上時間這個維度，以變成一個面向資料串流的開發典範，而這個發射事件的主體，在不同的 FRP 框架各由不同角色擔當， ReactiveCocoa 是 Signal，RxSwift 是 Observable，Combine 就是 Publisher，今天的主角會是 RxSwift，不過 Combine 與它的概念幾乎都是相通的，物件間的對應關係也幾乎是 1:1， 所以同樣的概念也完全適用。



Observable 的觀念一看 Rx 家族用來說明 Observable 的示意圖就能更容易地明白（[圖片來源](http://reactivex.io/documentation/observable.html)）
![observable](https://i.imgur.com/LJDykO8.png)



由此可知 Observable 可視為用時間為 index 的 event collection ，若是 collection 那麼我們所熟的 high-order function ，這些從 FP 來的概念也可以套用在 Observbale 上，因此 map、flatMap等這些熟悉的伙伴們都出現了，這即是 FRP 裡 F 的部分。

## 測試時間有點難

那今天的主題是什麼呢？我們知道 Observable 發出來的都是非同步事件，所以在測試用了 Observbale 的程式碼時，如果牽扯到 thread 的切換，那你就得使用非同步的測試方式 （比如 `XCTWaiter`），一個適當的測試應該 Observable 裡的事件來源其實都是用 mock 來取代真正的非同步工作，所以一定很快，可惜你仍然需要為這種測試方式設一個等待的時間；更困難的問題是時間是複雜的，如果我們想要更精準地驗確一個工作前開始、中間以及結束時的狀態就會把測試變得很複雜。但別忘了利用 time 當 index 的 event collection 這個概念，若我們能實做的話是不是就能控制時間，測試也能做得更明確嚴謹！而 RxSwift 就提供了 RxTest 這個附加的框架來幫忙這件事，就一起理解一下這件事是怎麼辦到的。

先看個例子，一個可以打 request 與有個 String 結果 Observable 的 ViewModel 

{% highlight Swift %}

struct Dependency {

  let request: () -> Observable<String>

  let scheduler: SchedulerType

}



class ViewModel {

  private let dependency: Dependency

  let result: Observable<String>

  private let resultRelay = PublishRelay<String>()

  private var bag = DisposeBag()

   

  init(_ dependency: Dependency) {

​    self.dependency = dependency

​    result = resultRelay.asObservable()

  }



  func doRequest() {

​    dependency.request()

​      .observeOn(dependency.scheduler)

​      .subscribe(onNext: { [weak self] in self?.resultRelay.accept($0) })

​      .disposed(by: bag)

  }

}

{% endhighlight %}

然後一般利用 waiter 的非同步測試會像：

{% highlight Swift %}

extension Dependency {

  static let mock = Dependency(request: {

​    return Observable.just("mock")

  }, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))

}



func testMockedRequest() {

  var results = [String]()

  let vm = ViewModel(.mock)

  vm.result

​    .observeOn(ConcurrentDispatchQueueScheduler(qos: .background))

​    .subscribe(onNext: { results.append($0) })

​    .disposed(by: bag)

   

  vm.doRequest()

  _ = XCTWaiter.wait(for: [XCTestExpectation()], timeout: 0.1)

  XCTAssertEqual(results, ["mock"])

}

{% endhighlight %}

## 把非同步變成同步

那接下來要怎麼控制時間呢？第一件事要把這些非同步的動作都變成同步的，我們才有可能控制它，變成一個以 time 為 index 的 event collection。要做到這件事需要先了解 RxSwift 裡的 `SchedulerType` 幹啥用的 （在 Combine 就是 `Scheduler`）？ `SchedulerType` 繼承自 `ImmediateSchedulerType`，共有一個 property `var now : RxTime { get }` 和三個 method ，分別是三種 schedule：立刻馬上做，延後一段時間再做與週期性地做，其實 scheduler 的功能就是要有個 now 做為時間軸起點的參考點，然後不同時間點對不同的事情做排程在不同的 thread 上。具體地來說 RxSwift 有  `MainScheduler` (main thread)，`DispatchQueueScheduler` （包裝 GCD）以及` OperationQueueScheduler` （包裝 OperationQueue）； 在 Combine 中，`DispatchQueue`，`OperationQueue`與 `RunLoop` 本身都遵循 `Scheduler` 了，更為靠近原本的用法習慣。



本文先只看第一種立刻馬上要做的 `func schedule<StateType>(_ state: StateType, action: @escaping (StateType) -> Disposable) -> Disposable`，這件事在以往的 callback 做法就會是

{% highlight Swift %}

myQueue.async {

​    doSomething()

}

{% endhighlight %}

而現在我們的目標是要能操控事件的發生時間，對於馬上要做的事件時間狀態只有做與不做兩種，所以解法就是放個 array 儲存要排程的工作，然後有個 method 可以觸放所有儲存起來，準備要發生的工作（注意以下先略過了其它兩種 schedule）



首先我們需要一個「包裝工作的容器」，而且它必須是個 Disposable ，因為你可以發現 schedule method 是回一個 Disposable

{% highlight Swift %}

final class VirtualSchedulerItem: Disposable {

  // 真正的非同步工作在此

  typealias Action = () -> Disposable



  let action: Action

  var disposable = SingleAssignmentDisposable()

  var isDisposed: Bool {

​    return disposable.isDisposed

  }



  init(_ action: @escaping Action) {

​    self.action = action

  }

  // 真正執行工作的觸發點

  func invoke() {

​    disposable.setDisposable(action())

  }



  func dispose() {

​    disposable.dispose()

  }

}

{% endhighlight %}

接下來要來做一個自己的 Scheduler：

{% highlight Swift %}

class MyTestScheudler: SchedulerType {

  private var scheduleds: [VirtualSchedulerItem] = []

  // RxTime 其實是 Date 的 typealias

  var now: RxTime



  init(now: RxTime = RxTime()) {

​    self.now = now

  }



  func schedule<StateType>(_ state: StateType, action: @escaping (StateType) -> Disposable) -> Disposable {

​    let compositeDisposable = CompositeDisposable()

​    // 把工作包在 scheduler item 裡

​    let item = VirtualSchedulerItem {

​      return action(state)

​    }

​    

​    scheduleds.append(item)

​    _ = compositeDisposable.insert(item)

​    return compositeDisposable

  }

  // 真正執行所有 scheduled task 

  func start() {

​    // 這裡就把所有工作都變成同步了

​    for item in scheduleds {

​      item.invoke()

​    }

​    scheduleds.removeAll() // 做完就可以移除了

  }

}

{% endhighlight %}



修改我們的 test：



{% highlight Swift %}

let testScheduler = MyTestScheudler()

extension Dependency {

  static let test = Dependency(request: {

​    return Observable.just("mock")

  }, scheduler: testScheduler)

}



func testMockedRequest() {

  var results = [String]()

  let vm = ViewModel(.test)

  vm.result

​    .observeOn(testScheduler)

​    .subscribe(onNext: {

​          results.append($0)

​       

​    })

​    .disposed(by: bag)

  // 以下才有變化

  vm.doRequest()

  XCTAssertEqual(results, []) // we can test the beginning

  testScheduler.start()

  XCTAssertEqual(results, ["mock"])

}



{% endhighlight %}

跑一下測試秒過，雖然是個簡單的例子，但不用等 expectation 就是爽啦！下集將介紹如果事情沒有要馬上發生的情況，以及若希望事件觸發也能指定發生在時間軸上的某一個點該如何進行。

