---
layout: post
title: 'Value Type 也會在 Heap 裡嗎？'
date: 2019-07-30
author: Pofat
color: rgb(60,219,192)
cover: 'https://i.imgur.com/I2QGC8O.jpg'
tags: Swift struct
---

最近和兩位志同道合的 iOS 發燒友（現在還有人用這詞嗎？）一起開辦了一個針對 iOS 開發者的中文 Podcast，第一集就榮幸地有大師來踢館，因為我們在節目裡提到 Swift 的 value types 都會在 stack 裡，大師 CJ 表示不對喔，若是 reference type 的 value property 或者被 closure 捕獲的 value type 都會在 heap 裡

<iframe src="https://giphy.com/embed/fdRLQ2ApeAYkaXqXML" width="480" height="480" style="display:block" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

這就有趣了不追究不行，但現在還沒有能力看完 Swift run time 的 source code，只好硬著頭皮想了個湊合的辦法來實驗一下。

## 指向 Class Instance 的 Header Pointer

以前看指標文章時有玩過直接取得位址後，可以對位址內容硬改值以改變本該無法被 mutate 的東西（比如 `let` ）
所以我們先宣告一個 class ，建立一個 instance，我們知道這個 instance 實際上的內容會存在 heap 裡的一塊連續空間裡，然後我們搞一個指向它 instance 開頭的指標

{% highlight Swift %}
class Person {
  let name = "Pofat"
  let age = 18
  let married = true
}

let pofat = Person()

let headerPointer = Unmanaged.passUnretained(pofat as AnyObject).toOpaque() 
{% endhighlight %}

照理來說這個 instance 裡所有的 property 應該會依著宣告的順序依序擺在這個 `headerPointer` 指向的位址之後，不過 class 前面有兩個 byte 是用來放 descriptor 以及 retain count，所以 property 會從第三個 byte 開始，我們用 `MemoryLayout` 來取得各個 property 所佔的空間大小，依序找出 `name`, `age` 以及 `married` 的位址

{% highlight Swift %}
let nameOffset = 16
let ageOffset = nameOffset + MemoryLayout<String>.stride
let marriedOffset = ageOffset + MemoryLayout<Int>.stride

// Bound to the type of each property
let namePointer = haderPointer.advanced(by: nameOffset).assumingMemoryBound(to: String.self)
let agePointer = headerPointer.advanced(by: ageOffset).assumingMemoryBound(to: Int.self)
let marriedPointer = headerPointer.advanced(by: marriedOffset).assumingMemoryBound(to: Bool.self)
{% endhighlight %}

這些位址當然還是在 heap 裡，而且應該指到就是這些 value type 的 property 所在地（吧）
我們直接強行改變這些位置上的值實驗看看

{% highlight Swift %}
namePointer.pointee = "Joe"
agePointer.pointee = 30
marriedPointer.pointee = false

print(pofat.name)       // get "Joe"
print(pofat.age)        // get 30
print(pofat.married)    // get false
{% endhighlight %}

OK，我們可以看到值如預料地變化了，這些 value type 真的是在 heap 裡啊！
也用了 Colab 做了一個可以直接在線上 run code 的[版本](https://t.co/oiTZvLTKDB?amp=1)

## 參考資料

1. weak self 第一集：[Swift API 設計之 Value Type 與 Reference Type](https://twitter.com/weak_self/status/1155511615661285376?s=21)
2. 酄迎訂閱13、喬喬和我共同製作的 Podcast ，這裡有[訂閱指南](https://t.co/dWSjXobT1l?amp=1)
3. CJ Lin 大大與他帶來的[文章](https://link.medium.com/v96sFdHDIY)
