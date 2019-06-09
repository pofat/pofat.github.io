---
layout: post
title: '重新檢視 Swift 的 Protocol (三）'
date: 2019-05-22
author: Pofat
color: rgb(253,117,36)
cover: 'https://i.imgur.com/rMFuHjT.jpg'
tags: Swift, Protocol
---

讓我們直接來到三部曲的終章，為了營造終局的高潮我塞了不少資訊在這篇文章裡，大家先喝點水伸展一下。

![really](https://i.imgur.com/rMFuHjT.jpg)

前篇提到了一個破天荒的想法，自己實做一個 Protocol Witness Table，所以我們先來深入點看看 Protocol Witness Table，首先是 compiler 在 SIL 階段做了什麼事（[官方文件](https://github.com/apple/swift/blob/d24bc387973bcf60f5596ff387b996c1eb356456/docs/SIL.rst#witness-tables)），首先文件裡提到：

> A witness table is emitted for every declared explicit conformance.

所以每當我們當聲明一個型別遵守(confrom to)某個 Protocol 時就會生成一個 witness table，而這個 witness table 會用「型別：Protocol」這組合生成一個獨特的 id 來當 key 保存著，文件裡共定義五種遵守 protocol 的可能性，我們只看最一般的一般性遵守

`protocol-conformance ::= normal-protocol-conformance`

因為每個型別只能遵守一個 protocol 一次（重點重點！），所以可以保證這樣的配對是 1:1，而且也只有這個一般的遵守方式會直接與 witness table 關聯，其它比較複雜的情況多半是間接或多個 table 之間的關聯。至於一個 witness table 會由 4 個 entry 組成以描述所有相關資訊，我們只在意最重要的一個 method entry：

`sil-witness-entry ::= 'method' sil-decl-ref ':' sil-function-name`

這裡很單純，前者是 protocol 所定義的介面，後者是真實的 method 實作，只是把它們連結起來而已，如這裡太抽象我們可以看一下圖和程式碼，就拿 WWDC 2016 [Session 416](https://developer.apple.com/videos/play/wwdc2016/416/) 的範例來說：

{% highlight Swift %}
protocol Drawable {
    func draw()
}

struct Point: Drawable {
    var x: Int
    var y: Int

    func draw() {
        print("Draw a point at (\(x), \(y))")
    }
}

// Main
let point: Drawable = Point(x: 1, y: 1)
point.draw()
{% endhighlight %}

我們這裡的變數 point 會是個 Existential Container，因為我們宣告它的型別是個 protocol，結合前述所說，container 和 protocol witness table 長得像這樣（大家還記得[第二集](https://pofat.dev/2019/05/21/%E9%87%8D%E6%96%B0%E6%AA%A2%E8%A6%96-swift-%E7%9A%84-protocol-%E4%BA%8C.html)講到 container 的模樣嗎？）

{% include image.html url="https://i.imgur.com/WwlbY0v.png" description="圖片來源：https://developer.apple.com/videos/play/wwdc2016/416/" %}
<br>

如果用程式碼表示，這個 point 的 container 與 Point: Drawable 的 witness table 會像這樣：

{% highlight Swift %}
struct ExistentialContainer {
    var valueBuffer: (Int, Int, Int)
    var vwt: UnsafePointer<ValueWitness>
    var pwt: UnsafePointer<ProtocolWitness>
}

struct ProtocolWitness {
    var descriptor: ProtocolConformanceDescriptor
    var draw: FunctionRef
}
{% endhighlight %}

懷疑論者至此如果還有懷疑，可以試著在前面的 point.draw() 打個中斷點，執行程式並停在該處，在 lldb 裡讀一下 register ，可以看到：

{% include image.html url="https://i.imgur.com/L07LC6v.png" description="注意 $rsi 暫存器" %}
<br>
其中 0x000000010fb21978 是 protocol witness table，我們檢查一下裡面的內容
{% include image.html url="https://i.imgur.com/YsP4ywG.png" description="" %}
<br>
第一個是指向 descriptor ，第二個就是指向我們 draw() 在 point 的實作，我們將第二個的位址反組譯一下，可以看到它的確就是 Point 對 Drawable 裡 draw() 的實作（在我程式裡是第 19 行沒錯）

{% include image.html url="https://i.imgur.com/EGbNYi8.png" description="" %}
<br>

探究至此，想必大家都已經有很具體的認知了，那我們想想，這樣的設計可能會有什麼問題？

除了前集提到的 PAT 不能生成 container 外，有時這 1對1 的關係（一型別只能遵守一個 protocol 一次）讓我們不能以最大效益共用程式碼，比如前一集提到 Request （忘記來[這裡](https://pofat.dev/2019/05/21/%E9%87%8D%E6%96%B0%E6%AA%A2%E8%A6%96-swift-%E7%9A%84-protocol-%E4%BA%8C.html)看），如果我們想要生成不同回傳型別的 Request ，就要宣告不同的 concrete type ，比如 UserRequest，ListsRequest等，其實大同小異，這樣就要宣告一個新型別好像有點浪費？

## 不如自己來實作 witness table 吧！

從前面 Point 和 Drawable 的例子來看，如果我們希望 Point 有不同的畫法呢？比如空心點、實心點或者虛線點，這樣勢必要三種型別才能達成啊，不管是純遵守 protocol 或者使用繼承 ！不過方才我們也看到 witness table 在 SIL 的結構，看起來不難搞，其實就是 draw 要保存一個實作的位址嘛！在 Swift 裡 function 可是 “first-class citizen”，也就是 function 可以當作變數來運用，這提供了一個大好的契機，我們可以把 Drawable witness table 的 draw property 改寫一下

`var draw: FunctionRef -> var draw: (Shape) -> ()`

整個例子就可以改寫成

{% highlight Swift %}
// From:
protocol Drawable {
    func draw()
}

// To:
struct Drawing<Shape> {
    var draw: (Shape) -> ()
}

// 用來畫實心點
let solidPointDrawer = Drawing<CGPoint> { point in
    print("draw solid point: (\(point.x), \(point.y))")
}

// 用來畫空心點
let hollowPointDrawer = Drawing<CGPoint> { point in
    print("draw hollow point: (\(point.x), \(point.y))")
}                      

let point = CGPoint(x: 1, y: 1)

solidPointDrawer.draw(point) // 畫實心點
hollowPointDrawer.draw(point) // 畫空心點
{% endhighlight %}

是不是突然就有走入新世界的感覺了呢？打鐵要趁熱，我們繼續把上次的 network rqeust 範例一起用這種方式重新改寫（因為要傳入 handler 且它是個 closure，如果要寫得優雅點需要用 curry 的方式消參數，怕失焦就先借用 RxSwift 該程式碼優雅點）

{% highlight Swift %}
import Foundation
import RxSwift

// MARK: Model
struct User: Decodable {
    let id: Int
    let name: String
}

let usersDecode: (Data) -> [User]? = { data in
    return try? JSONDecoder().decode([User].self, from: data)
}

// MARK: Request related
enum HTTPMethod: String {
    case GET
    case POST
}

struct Request<Response: Decodable> {
    var endpoint: String
    var method: HTTPMethod
    var param: [String: Any]?
    let decode: (Data) -> Response?
}

struct RequestSender {
    let base = "https://foo.bar"
    private let sendingRequst: (URLRequest) -> Observable<Data> // To abstract your network framework
    init(_ sendingRequst: @escaping(URLRequest) -> Observable<Data>) {
        self.sendingRequst = sendingRequst
    }

    func send<Response: Decodable>(_ r: Request<Response>) -> Observable<Response?> {
        let url = URL(string: base + r.endpoint)!
        var request = URLRequest(url: url)
        request.httpMethod = r.method.rawValue
        if let param = r.param, let body = try? JSONSerialization.data(withJSONObject: param, options: .prettyPrinted) {
            request.httpBody = body
        }

        return sendingRequst(request).map(r.decode)
    }
}

// MARK: Main
// Instance of requests and request sender
let getUsersRequest = Request(endpoint: "/users", method: .GET, param: nil, decode: usersDecode)

// You can use any other network framework such as Alamofire
let urlSessionRequestSender = RequestSender { request in
    return Observable<Data>.create { subscriber in
        let task = URLSession.shared.dataTask(with: request) { data, res, error in
            if let data = data {
                subscriber.onNext(data)
            } else {
                subscriber.onError(error!)
            }
        }
        task.resume()
        return Disposables.create()
    }
}

// Actually do request (get users)
_ = urlSessionRequestSender.send(getUsersRequest).subscribe(onNext: { users in
    guard let users = users else {
        return
	}

    print(users)
})
{% endhighlight %}

RequestSender 的部分就寫個意思意思（但真的會動），大家可以再自己優化，主要的重點還是在 Request 這個泛型 struct 裡，利用它的泛型資訊再讓 RequestSender 可以處理各種 return type 的 request。

以上，protocol 再檢視三部曲到此結束，希望對大家有幫助，接下來請大家期待第一集（疑？）

## 後記與參考資料

自己實作 witness table 的想法來自於於 Brandon Willams 的[演講](https://www.fewbutripe.com/talks/#protocol-witnesses)（推薦大家值得一看的影片），這個想法實在太酷炫，我一知道後立刻興奮得把自己手邊的 protocol 範例實作一遍，在此感謝大神的指教讓我走入新世界。
