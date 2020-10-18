---
layout: post
title: 'The Journey Of Swift Concurrency - Atomics'
date: 2020-10-18
author: Pofat
color: rgb(253,117,36)
cover: 'https://i.imgur.com/5DnnuY5.png'
tags: Swift Concurrency
---



由於在 weak self [56集](https://weakself.dev/episodes/56)聊到了這陣子 Apple 一周一 framework，蓬勃地產出~~Bug~~，我認為這個 [Swift Atomics](https://github.com/apple/swift-atomics) 是整個 Swift Concurrency 大業的一個分支，之後會陸續就我的看法，將  Swift Concurrency 的起頭，以及目前已經實現的部分寫文探討。本來 Atomics 應該是整個系列的第三或第四集，但因為 Podcast 節目要出了，迫在眉睫、褲子著火，只好先聊 Atomics （疑又是第三集先出啊)，也放棄了 Integer 型別的系列 index，以增加~~惡補的可能性~~彈性。

## 為何要有 Swift Atomics?

相信 iOS 開發者都很熟悉，如果要在不同的 thread 間讀寫共享的變數，特別是 value type 如 `Array` 或 `Dictionary`，我們都會用個 GCD queue 限制讀寫都只在同一個 queue 上或者用 `NSLock` 之類的 lock 來保證讀寫順序與完整性。

然而對 Swift 而言，希望能成為系統層級可使用的語言以及具有更高層次抽象的 Concurrency 做法（比如 Coroutines），需要更底層的界面讓開發者能夠直接宣告一個可以安全訪問的同步結構。

簡而言之就是你能宣告一個型別，就可以直接在不同的 thread 間讀寫它而不會出事，你不需要再去考慮該用 GCD 或 lock 來保證它的安全性，畢竟這兩種方法都有它的缺點與限制：

使用 GCD，當數據或對象增加時，容易變得複雜，你很難一下明白是「哪一個 queue」在保護「哪一個數據」； 有的情況使用 NSLock根本無用，比如在一個 mutating method 裡放 lock，mutating 本身實作概念是會先 copy 一份出去修改，再寫回原本的變數中。除以此外，lock 的做法可能還會有效能的疑慮。

所以，Swift 應該搞個自己的東西讓我們直接用才對，基本上目標會是：

* 一個新的類型，會在語義上明顯地標示這是一個 atomic 操作，最要緊的就是 concurrent mutation，一個 Swift 以前沒有的概念。
* 操作時要明確地給予 memory order 參數，以明白讀寫共享的 memory 時順序是如何地保證。

當然如同介紹與提案說的，這是個滿是坑的領域，跟 `Unsafe-` 家族一樣，希望少用更或者別用。這裡只是實現了真正的 concurrent mutation 的可能性（因為以前是不可能的），實質上 Atomics 物件可以視作 `UnsafePointer` 的 wrapper 而已，用起來就是怕。

實際上做的事大概是兩部分：

1. 與 C/C++ 相似的同步記憶體模型以及橋接 C/C++的記憶體順序
2. 基於 C Atomic 實現核心功能與 class-based 的 Atomic 物件給 Swift 開發者使用

## Swift Atomics 為何是個 Package

Atomics 的提案是 [SE-0282](https://github.com/apple/swift-evolution/blob/main/proposals/0282-atomics.md)，然而這個提案是經過翻修的，原本從底層要做的橋接 C++/C 之記憶體模型到開發者可以建立的 Atomic 類型都有定義。這個目標原本是希望在，不過在考慮對於 API/ABI 穩定的影響，最後在 [Joe 的建議](https://forums.swift.org/t/se-0282-low-level-atomic-operations/35382/59)下，這個提案修改為針對記憶體模型的改進和與此相關的底層 api，而最上層的 class-based 的 Atomics 物件則用 package 來推出。

關於第一版的提案可見[此](https://github.com/apple/swift-evolution/blob/3a358a07e878a58bec256639d2beb48461fc3177/proposals/0282-atomics.md)，推友[四娘](https://twitter.com/kemchenj)有針對原始版寫了一篇[譯文](https://kemchenj.github.io/2020-10-02/)，英文苦手可參考。

## 怎麼用

最常見的 thread-unsafe access 例子是

{% highlight Swift %}
import Dispatch

struct Counter {
    private var value = 0
    mutating func increment() {
        value += 1
    }
}

DispatchQueue.concurrentPerform(iterations: 10) { _ in
    for _ in 0..<1000 {
        counter.increment()
    }
}

print(counter.value) // 會很奇怪不是 10_000
{% endhighlight %}

如果用 Atomics 就會得到正確的結果

{% highlight Swift %}
import Atomics
import Dispatch

let counter = ManagedAtomic<Int>(0)

DispatchQueue.concurrentPerform(iterations: 10) { _ in
  for _ in 0 ..< 1000 {
    counter.wrappingIncrement(ordering: .relaxed)
  }
}
counter.load(ordering: .relaxed) // ⟹ 10_000
{% endhighlight %}

## 支援的型別

* 各類整數們： `Int, Int64, Int32, Int16, Int8` 與 `UInt, UInt64, UInt32, UInt16, UInt8`
* `Bool`
* 標準的 Pointer 型別： `UnsafeRawPointer, UnsafeMutableRawPointer, UnsafePointer<T>, UnsafeMutablePointer<T>, Unmanaged<T>`，以及它們的 Optional 型態
* 任何 `RawValue` 是 atomic type 的 `RawRepresentable`，比如

{% highlight Swift %}
enum AtomicEnum: Int, AtomicValue {
    case foo, bar
}

let aEnum = ManagedAtomic<AtomicEnum>(.foo)
{% endhighlight %}

* Reference type 要 confrom AtomicReference protocol 才行，如一個 LinkedList 裡最常用的 Node 結構

{% highlight Swift %}
class Node: AtomicReference {
    let next: ManagedAtomic<Node?>
    var value: Element?
}
{% endhighlight %}

## 對現有的衝擊

上面的 concurrent DispatchQueue 例子顯然違反了當前的 Swift memery exclusive access rules ，不能同時有 read/write 或 write/write access，所以這個規則被改寫了為 `除了 read 或 atomic access，不得有在同一個 variable 上有重複的 access`，然後 atomic access 就是我們可以使用的這些 api 如 `func load(ordering: AtomicLoadOrdering) -> Value` 等

至於記憶體管理，目前提供了兩種形態的 class，當然還是鼓勵大家用 ARC 的版本，除非你完全知道自己在幹嘛

* `ManagedAtomic<T>`： 仰賴 ARC 管理記憶體
* `UnsafeAtomic<T>`： 顧名思義，你要手動自己管理記憶體，可視為 UnsafePointer 再包一層

這兩種物件都提供了六種可用的操作

{% highlight Swift %}
func load(ordering: AtomicLoadOrdering) -> Value
func store(_ desired: Value, ordering: AtomicStoreOrdering)
func exchange(_ desired: Value, ordering: AtomicUpdateOrdering) -> Value

func compareExchange(
    expected: Value,
    desired: Value,
    ordering: AtomicUpdateOrdering
) -> (exchanged: Bool, original: Value)

func compareExchange(
    expected: Value,
    desired: Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
) -> (exchanged: Bool, original: Value)

func weakCompareExchange(
    expected: Value,
    desired: Value,
    successOrdering: AtomicUpdateOrdering,
    failureOrdering: AtomicLoadOrdering
) -> (exchanged: Bool, original: Value)
{% endhighlight %}

基本上這個操作和 C/C++ 是相似的，且 memory ordering 更是完全 1:1 對應到 C/C++ `std::memory_order` 的定義。

## Lock-Free 但不是 Wait-Free

Swift Atomics 保證用這物件可以完全不會被擋住 (non-blocking)，但不代表一定不用等，實際上的 atomic 操作會因對應的執行平台而異，如果是 compare and exchange loop 的話，你可能會因為不同的排程策略而需要等比較久才能完成操作，compare and exchange 的概念可能可以這樣大略表示

{% highlight Swift %}
func compareAndSwap(old: Value, new: Value, current: Value) -> Bool {
    guard old == current else {
        return false
    }
    old = new // assgin new value to real address
    return true
}

// The loop
repeat {
    let oldValue = value.load() // fetch old value
    let newValue = newValue // value we want to set
    let success = compareAndSwap(old: oldValue,
                                 new: newValue,
                                 current: value.load()) 
} while !success

{% endhighlight %}

## App 開發者該怎麼做？

基本上這個 pakcage 可能絕大多數的 iOS 開發者是完全用不到的，至少要到 [SwiftCoroutine](https://github.com/belozierov/SwiftCoroutine) 這樣程度的 API 才有可能被大家舒服地使用的，或許有興趣的人可以加入幫忙處理一些底層未實現的功能，或著試著用它搭建更高層可以給一般 app 開發者使用的套件。
