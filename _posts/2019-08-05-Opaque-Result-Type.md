---
layout: post
title: 'Opaque Result Type'
date: 2019-08-05
author: Pofat
color: rgb(206,206,206)
cover: 'https://cdn.pixabay.com/photo/2016/03/09/15/26/ruler-1246653_1280.jpg'
tags: Swift protocol SwiftUI
---

上次寫完 PAT (Protcol with Assocaited Type) 不能做為型別的另一種解法沒多久（請見[重新檢視 Swift 的 Protocl（二）](https://pofat.dev/2019/05/21/%E9%87%8D%E6%96%B0%E6%AA%A2%E8%A6%96-swift-%E7%9A%84-protocol-%E4%BA%8C.html)），就迎來 WWDC 2019，在萬眾矚目的 SwiftUI 裡立刻展現了官方解決這個問題的帥氣手法，`some View`，也適逢 Podcast 提問箱有人提問什麼是 [Opaque Result Type](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) ，本期 Podcast 只有點到沒有深聊，所以在這討論一下。

## 為何要有 Opaque Result Type

首先了解一下這個特性要解決什麼 Swift 5.1 前無法解決的問題：
1. 型別上對返回類型的抽象化無法被實現 (Type-level abastraction of returns)
2. 無法建立帶有泛型類型的 existentail container （也就是無法用 PAT 做為一種類型）
3. 如果有巢狀結構或是那種多層的泛型使用時常會需要在宣告時指明多層又臭又長的各層類型，像是：

{% highlight Swift %}
protocol Shape {}

struct Rectangle: Shape {}

struct Union<A: Shape, B: Shape>: Shape {
    var a: Shape
    var b: Shape
}

struct Transformed<S: Shape>: Shape {
    var shape: S
}

protocol GameObject {
    associatedtype ShapeType: Shape
    var shape: ShapeType { get }
}

struct EightPointedStar: GameObject {
    var shape: Union<Rectangle, Transformed<Rectangle>> {
        return Union(a:Rectangle(), b:Transformed(shape: Rectangle()))
    }
}
{% endhighlight %}

可見最後 `EightPointedStar.shape` 的定義是多麼地可怕，以上幾個問題其實都來自於一個原先 Swift 泛型系綂設計的不足，那就是:

> 只能由呼叫方(caller)來決定被呼叫方(callee)泛型參數最終要填入的真正型別

我們回顧一下實例：

{% highlight Swift %}
protocol Shape {
    func draw() -> String
}

struct Triangle: Shape {
    var size: Int

    // 會印出：
    // *
    // **
    // ***
    func draw() -> String {
        var result = [String]()
        for length in 1...size {
            result.append(String(repeating: "*", count: length))
        }
        return result.joined(separator: "\n")
    }
}

struct FlippedShape<T: Shape>: Shape {
    var shape: T
    func draw() -> String {
        let lines = shape.draw().split(separator: "\n")
        return lines.reversed().joined(separator: "\n")
    }
}

let smallTriangle = Triangle(size: 3)
let flippedTriangle = FlippedShape(shape: smallTriangle) // 這裡由 FlippedShape 決定 T == Triangle
print(flippedTriangle.draw())
// ***
// **
// *
{% endhighlight %}

所以 Opaque Result Type 最重要的精神就是讓被呼叫方（callee），也就是實作方決定要回什麼真實的型別；因此外部呼叫後得到的就會是一個抽象的型別，這點與先前在宣告時給參數，外部需要明確地給予一個決定性的真實型別相反，可以視作是反向的泛型（reverse-generic）。

有了這個特性，我們前面的 `EightPointedStar` 就可以簡化成

{% highlight Swift %}
struct EightPointedStar: GameObject {
    var shape: some Shape {
        return Union(a:Rectangle(), b:Transformed(shape: Rectangle()))
    }
}
{% endhighlight %}

至於 PAT，我們就拿 SwiftUI 裡的 `some View` 來說明，基本上 `View` 是一個帶有 associated type 的 Protocol

{% highlight Swift %}
public protocol View {
    associatedtype Body : View

    var body: Self.Body { get }
}
{% endhighlight %}

使用上是這樣的

{% highlight Swift %}
struct ContentView: View {
    var body: some View {
        Text("Hello World")
    }
}
{% endhighlight %}

如此一來實作 `View` Protocol 的 ContentView 可以自由地決定要生成什麼 view，再也不是靠外部決定了，也因此 SwiftUI 在 runtime 呼叫 `body` 時無法跟我們以前的使用習慣一樣，它看到的是一個抽象的型別，要用遞迴的方式逐一檢查所有的 view 到底有哪些具體型別，這其中也有用 [metadata](https://forums.swift.org/t/dynamicviewproperty/25627/4) 來幫忙解析這些資訊。

## 限制與注意事項

* 最終的具體型別要在 compile time 能確定，比如你不能寫出

{% highlight Swift %}
protocol P { }
extension Int : P { }
extension String : P { }

func f1(condition: Bool) -> some P {
    if condition {
        return 1
    } else {
        return "opaque"   // compile error
    }
}
{% endhighlight %}

* 最終還是會對應到一個具體的型別的，且具有單一性

{% highlight Swift %}
func foo<T: Equatable>(x: T, y: T) -> some Equatable {
    let condition = x == y // 合法, 因為 x、y 是同一個泛型參數
    return condition ? 1738 : 679
}

let x = foo("apples", "bananas")
let y = foo("apples", "some fruit nobody's ever heard of")

print(x == y) // 也合法，因為同個泛型參數
{% endhighlight %}

以下做法是不合法的

{% highlight Swift %}
func makeOpaque<T>(_: T.Type) -> some Any { /* ... */ }
var x = makeOpaque(Int.self)
x = makeOpaque(Double.self) // compile error，因為 x 的型別應該是 Any<Int>
{% endhighlight %}

* 不可包在 Optional 中，不可宣告在 protocol 中

以下的寫法也都不行，compiler 會抱怨的！

{% highlight Swift %}
func f(flip: Bool) -> (some P)? {
  // ...
}

protocol Q {
    func f() -> some P
}
{% endhighlight %}


## 參考資料

* weak self 第二集：[原來我不會用 protocol](https://twitter.com/weak_self/status/1158049319653560321)
