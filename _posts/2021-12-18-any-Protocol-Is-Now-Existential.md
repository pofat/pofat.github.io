---
layout: post
title: 'any Protocol Is Now Existential'
date: 2021-12-18
author: Pofat
color: rgb(253,117,36)
cover: 'https://i.imgur.com/5DnnuY5.png'
tags: Swift Protocol
---


在 Swift 3 後， source code breaking change 已經少有， 最近有個 [evolution](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md) 可能會引來部分的 breaking change 卻依然值得， 那就是用 protocol 標識為一個物件的型別時， 需要加上 `any` 的前綴， 比如原本是`let foo: FooProtocol = FooConformance()` 會改成  `let foo: any FooProtocol = FooConformance()`。

## 萬惡 PAT

把 protocol 這樣使用， 其實並不是在告訴 compiler `foo` 這個變數是 `FooProtocol` 這個型別， 而是讓 compiler 建了一個 existential container，但使用上和指示型別一模一樣，這樣引起了不少混淆， 因為它們其實代表的意義不同， 況且如果 protocol 裡有 `Self` 或`associatedtype` 在 [SE-309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) 之前也不能這樣用， compiler 會跟你抱怨 ` XXX can only be used as a generic constraint because it has "Self or associated type requirements"`，常常讓使用者搞混情況和撞牆，常見的解法是照 standard library 裡面教我們的 type erasure ， 做出如同 `AnyHashable` 之類的包裝來使用， 我們在以前的[文章](https://pofat.dev/2019/05/21/重新檢視-swift-的-protocol-二.html)有討論過。 

總之，雖然 Swift 帶給我們 Protocol Oriented Programming 的思維， 也允許我們在只需要關心變數間的值時不需要在乎它們之間的型別關係，同時卻也帶來了混淆和多種限制， 曾見大膽發文說要是 Swift Protocol 不能全部都可做 existential container 就不是真正有用啦！

## 解放

所以 [SE-309 Unlock existentials for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) 總算不負眾望地解決了這件事，實作的思想大致上可以說以前開發者做 type erasure 的工作由 compiler 來幫忙完成， 連因為有 Self 或 associated type 而不能建立existential container 這個臭名昭彰的 compile error 都被[拿掉了](https://github.com/apple/swift/pull/33767/files#r640209183)，可喜可賀啊（我也一樣會想念這個訊息的）！現在我們可以這樣寫了：

{% highlight Swift %}
protocol Copyable {
  func copy() -> Self
}

func test(c: Copyable) {
  let x = c.copy()
}
{% endhighlight %}

用一張圖來表示整個 evolution 就是 （[出處](https://twitter.com/jckarter/status/1453397244334329856?s=20)）
![Protocols are the same](￼￼https://pbs.twimg.com/media/FCuBFB-UcAkbYL5?format=jpg)

Protocol 的束縛至此完全被解放，不過使用上混淆的狀況仍未解決， 比如從目前使用語法上的角度來看， `func foo(s: Sequence)` 與 `func foo<S: Sequence>(s: S)` 幾乎感覺不出差異， 其實 existential type 和 generic parameter 本意上是大不相同：前者的本意是做為值層面的抽象 (value-level abastraction），你可以只關注它的值而不用理解型別之間的關係， 型別資訊等於被抺除了；而後者則是類型的抽象。 

## 語法影響想法

一旦 SE-309 出來， 大家能夠更輕鬆地寫出 existential container （因為語法也沒變， 你可以用的場景變多而已）， 這固然是好事， 不過也可能造成過度濫用， 只因這樣寫實在太輕鬆了，可能會導致大家心智模型被其簡便性引誘， 而在「該權衡抽象與否或何種抽象的時候」直接略過思考而採用 existential container，這就是 [SE-335 Introduce existential any](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md) 的任務， 在**語法層面上就做出區別**，比如接續上面的例子， 我們可以寫出 `let copy: any Copyable = SomeConformance()`。

雖然語法上有 breaking change ，但這個改變相對單純，是能夠做出migration tool 來自動轉移的， 為了緩和這個過程，目前打算 Swift 5.6 開始可以接受  `any` 的寫法， 在 Swift 6 才強制要在 existential type 加上 `any` ，值得一提的是這個語法可以使用在 typealias 或 associated type 裡：

{% highlight Swift %}
protocol P {}

protocol Requirements {
  associatedtype A
}

struct S1: Requirements {
  typealias A = any P // okay
}
{% endhighlight %}

## Metatype

除此之外還會影響到 metadata type的寫法， metadata 指的是用來描述 type 的相關資訊， 比如某 struct 裡有什麼 property ，其類型是 struct 等， 先回憶一下現有相關的語法：
* `.Type` ：即 Swift metadata 的 Type，比如 `Int.Type` 就是 Int 它 metadata 的型別；比較特別的是 protocol， 同樣的寫法並不是指 protocol 的 metatype 而是任何 conform 該 protocol 的 metatype ，Apple 稱其為 **existential metatype**
* `.self`：靜態的 metatype (相對的是使用 `type(of:)` 來取得動態的 metatype），舉例說明： `let intMetatype: Int.Type = Int.self`
* `.Protocol` ：如果我們需要的是 **protocol 本身的 metatype**，則要使用 `MyProtocol.Protocol` 來取得

引入 any後， 假如我們有一個 `protocol Q {}` ，上述各表達方式會變成：
* `Q.Type` -> `any Q.Type` ，因為它是 existential (any) metatype (.Type)
* `Q.self` -> `(any Q).self`
* `Q.Protocol` -> `(any Q).Type`

## 額外討論

`Any` 和 `AnyObject` 其實也都是 protocol， 最常見的使用情境都是用做 existential container ， 基於 evolution 的討論， 如果要讓其語法更一致的可能要將其各改名為 `any Value` 與 `any Object` ， 只是這會造成更大範圍的 breaking change ，而這兩個特殊存在想必大家也都用得很習慣， 對它的心智模型也沒有歪到哪去， 目前仍傾向保留現況。 

另外就是更簡潔的 protocol with associated value extension 寫法

{% highlight Swift %}
extension Array {
  mutating func append(contentsOf newElements: Sequence<Element>)
}
{% endhighlight %}

至此， Swift protocol 的最後一哩路也算是終於打通了吧！