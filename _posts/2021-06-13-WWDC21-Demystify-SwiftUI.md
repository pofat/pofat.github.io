---
layout: post
title: '[WWDC21] Demystify SwiftUI'
date: 2021-06-13
author: Pofat
color: rgb(6,142,255)
cover: '/assets/SwiftUI.png'
tags: Swift SwiftUI
---

WWDC 21 的 Keynote 看來格外平淡，但沒想到 session 公佈後卻意外地精彩絕倫，各種新東西、解密以及實務探討，我一眼就看到今年最不容錯過的 session - [Demystify SwiftUI](https://developer.apple.com/videos/play/wwdc2021/10022/)，姑且翻為 SwiftUI 解密，事後看完雖然沒有解多少密，但該注意的使用方式以避免效能或穩定性問題都有點出來。而且今年首度舉辦 Digital Lounge ，報名後一群開發者會被拉入 Slack，第二三天都有一起看的活動，其中第三天一起看的就是 SwiftUI 解密，重點是該 session 的講者也同時在 Slack 裡面即時回答大家的問題，事後也有留時間給大家額外提問、討論，在那裡又獲得不少寶藏啊！

相信此 session 絕不會在接下陸續發佈的 WWDC 精選解說文中缺席，這篇文就作為我個人的學習筆記，其中包含了 lounge 裡的討論與提問，和各位分享。

## Identity

當有人和我討論 Swift 裡 value 與 reference type 選擇的準則是什麼，我的回答都是：「在一般情況下，以需要 identity 與否做為要不要用 reference type 的依據」。在 SwiftUI 中，我們知道 View 都是 value type ，而 identity 則是給 SwiftUI 使用的。對於一個 render view 的 framework，需要知道它要不要更新「這個」 view ，所以身份識別是必要的。識別的方式分別為開發者主動給予識別資訊的 `Explicit identity` 與 SwiftUI 從程式碼結構中取得的 `Structural identity`

補充：基本的元件如 Text Button 在 runtime 用 lldb 觀察可得知都是一個 `UIView` 的 subclass `SwiftUI.DisplayList.ViewUpdate.Platform.CGDrawingView`

{% highlight shell %}
(_TtCOCV7SwiftUI11DisplayList11ViewUpdater8Platform13CGDrawingView) $46 = {
  _TtC7SwiftUI15_UIGraphicsView = {
    UIView = {
      UIResponder = {
        NSObject = {
          isa = SwiftUI.DisplayList.ViewUpdater.Platform.CGDrawingView
        }
      }
    }
  }
}
{% endhighlight %}

### Explicit Identity

為了讓 compiler 可以約束何時由開發者提供身份資訊 (identifier)，因此定義了 `Identifiable` 這個 protocol ，它要求你提供一個名為 `id` 的 property，且此 property 必需為 `Hashable` ，不難想像這些東西會被 SwiftUI 放在 Dictionary 或 Set 這樣的集合中以在 O(1) 的時間複雜度下查找，實例如下：

{% highlight Swift %}
var colors: [Color] = [.black, .white, .blue]
VStack {
    ForEach(colors, id: \.self) { color in
        Text(color.description)
    }
}
{% endhighlight %}

值得注意的是，如果你直接送給 `ForEach()` 一個動態長度的 collection 且不給 ID 會得到 warning （以前會是個 error ）；另外你也可以使用 `.id(xxx)` 來主動給某個 view identifier，如 `Text("headerSection").id(headerID)`

### Structrual Identity

透過 `@ViewBuilder` 這個 annotation ，可以在 compile time 知道有哪些 view 的組合，這也是若你去觀察 SwiftUI 的型別會發現很難閱讀的原因，因為都是一堆泛型的組成，這是在安全與效率下的 trade off ，比如最常見的 if-else branch :

{% highlight Swift %}
var body: some View {
    if someCondition {
        FooView()
    } else {
        BarView()
    }
}
// 以上會被轉成如下

some View = _ConditionalContent<FooView, BarView>()
{% endhighlight %}

注意 body 本身自帶 ViewBuilder ，只是沒有明示。

這樣的做法讓 view 在 runtime 有更多資訊知道誰是誰，畢竟「改變一個 view 的狀態」和「轉換成另一個 view」是兩種截然不同的成本。由此可見 `AnyView` 這樣的 type-erasure view 是要謹慎使用的，因為 SwiftUI 並無法在 compile time 確定 AnyView 背後到底是誰， Apple 工程師給出使用 AnyView 時機的準則是：「盡量用在包極少或完全不會變化的 view 上，若使用在 state 頻繁改變的場景可能引起 performance issue」。

### Identifier, State, View and Depedencies

首先 view, state 與 identifier 的關係是：
1. View 的 lifetime 由 identifier 決定
2. State 的 lifetime 會跟著 view 同生同死
3. 每當 identifier 開始出現，SwiftUI 會 allocate 一塊記憶體用來存放 state (即 `@State` 與 `@StateObject`)
4. 上述的記憶體由對應的 identifier 來做 index 
5. 在整個 lifetime 中，state 可能會被改變，state 的前一組值會保存下來用來比較是否有改變，不同的話才會重新 render
6. 有點神奇的是你的 `@State` 不用是 Equatable ，SwiftUI 也能理解值有沒有變化，估計是在 runtime 時利用 Metadata 去挖 properties 做比較
7. 既然 state 是要被保存的，所以會在 heap 裡

Dependencies 是從 view 外部來決定需不需要重 render 的準則，一旦變化了，會先重建該 view 的 body ，然後再 recursivley traverse 其所有 childe view 。SwiftUI 會利用 identity 與其 lifetime 來建立一個 dependency 的 directed acyclic graph (但從 code level 看整個 view 是個 tree)，如此的做法可以避免過度的 render ，比如 view 的結構是 A -> B -> C，當唯 A C 共享的 dependency 改變時，僅 A C 會被 render ，而 B 不會。但其實當你若有把 property 從 A 傳到 B ，那也代表有 dependency ，所以常見情況應該都還是會全部都有 dependency 因而全都 re-render。

至於 dependency cycle 是不被允許發生的，一旦發生了會被 system trap，可以理解這個檢查是發生在 runtime 。

### 來自 Apple 工程師的愛，新的 debug tool

說這麼多，我們又不是 compiler 也不是 SwiftUI，我只是想知道到底什麼時候 view 被重畫，又為什麼會被重畫而已啊！你的心聲 Apple 工程師[聽到了](https://twitter.com/luka_bernardi/status/1402045202714435585?s=20)，現在只要加上 `Self._printChanges()` 在你的 content view 裡，就能夠看到 identity、dependency 與 state 是否有被設值以及是否有變化（以上任一變化都會發生 render），具體使用方式如下：

{% highlight Swift %}
var body: some View {
    let _ = Self._printChanges()
    // ... your views ...
}
{% endhighlight %}

或者打個 breakpoint 在 body 裡並新增 action `po Self._printChanges()`

![breakpoint_example](/assets/breakpoint_printChanges.png)
如此一來當發生變化時你會在 console 中看到

{% highlight shell %}
ContentView: @self, @identity, _someState changed.
{% endhighlight %}

以上例子代表該 view 的 dependency （外部餵進 self 的參數）、identity 以及 state 都發生了變化，其實這裡就是第一次初始化後印出來的啦。

### 總結設計注意事項

可以知道 identifier 這東西大大地決定了整個 view render 的時機，lifetime 與 state 的 lifetime ，誤用時可能造成災難，因此要注意兩種不同 identifier 的使用方式： 

**Explicit identifier 需要唯一且穩定**

不同的 id 就會被視為不同 view 了，除了 overhead 外還有可能導致狀態的遺失，比如我們用 if else 控制同 view 的不同狀態

{% highlight Swift %}

if dayTime {
    FooView()
} else {
    FooView()
        .nightTimeMode()
}
{% endhighlight %}

其中， if 裡的 `FooView` 和 else 裡的會被視為兩個不同的 view ，當 dayTime 為 true ，會建立一塊記憶體保存其 state （稱為 state F)，當 dayTime 變  false，則又要了另一塊記憶體保存狀態 (state F')，當又回到 true branch 時，又一個新的記憶體建立（ state F''），是故你的狀態可能因此遺失了。

另外常見的可能誤用是以 collection 的 index 做 id，這是不穩定的 id ，因為如果你插入一個元素，其往後的 index 全都會變化（+1），SwiftUI 自然也會把它們視為新 view 重新 render 和 reset state 了；若使用 hash value 也要注意你使用的 hash 是不是會同值但卻有不同的 hash value 。

**使用 structural identifier 時先考慮能否換成 inert modifier**

這個應易懂：

{% highlight Swift %}

// prefer

FooView().opacity(condition ? 1 : 0.3)

// Do not

if condition {
    FooView().opacity(1)
} else {
    FooView().opacity(0.3)
}
{% endhighlight %}

總之就是減少無意義的 branch 或用 inert modifier 代替，因為它成本極低。不過問題來了，哪些 modifier 才是 inert ？有在 lounge 提問，但這題並沒有被回答到，session 中只舉了三個例子： `opacity()` 、 `padding(0)` 與 `transformEnvironemt(...) {}`，但其它的也不知道是還不是，這裡還有望知道的人指點。

### 其他有趣的討論

1. SwiftUI 不做 view diff ，只要 dependency 改變就會重 redner 
2. 與 `@State` 與 `@StateObject` 不同，`@ObservedObject` 的 lifetime 由 developer 掌控，它可以存活在任何地方
3. Group 很便宜，能用盡量用
4. 在 office hour 的小題目，以下例子中，可以使用 `@State` 、`@StateObject` 或 `@ObservedObject` 哪些？為何？

{% highlight Swift %}

class AppState: ObservableObject {
    static let shared = AppState()
    var manualPauseOfSpeaking: Bool = false
}

struct SceneTitleView: View {
    @StateObject var appState = AppState.shared
	// ...
}
{% endhighlight %}