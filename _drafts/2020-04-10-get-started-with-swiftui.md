---
layout: post
title:  "SwiftUI初体验"
date:   2020-04-10 09:02:10 +0800
tag: SwiftUI
---

WWDC19最令人就是Apple发布了`SwiftUI`，尽管处于非常早期的一个版本。

## 生命周期

UIKit中 UIViewController有生命周期的方法，我们可以在对应的方法中处理一些特殊的逻辑。


```swift
List {

}
.onAppear {
    
}
.onDisAppear {

}
```

### 容器


| SwiftUI | 描述 |
| --- | --- |
| NavigationView | |
| TabView | |
| Group | |


### 输入

| SwiftUI | 描述 |
| --- | --- |
| Toggle | 开关 |
| Button | 按钮 |
| TextField | 输入框 |
| Slider | |
| DatePicker | |
| Picker | |
| Stepper | |
| Tap | 点击，有单击与双击 `.onTapGesture(count: 2)` |

### 布局

| SwiftUI | 描述 |
| ---  | --- |
| HStack | 横向布局的 UIStackView |
| VStack | 纵向布局的 UIStackView  |
| ZStack |  |
| Background | 增加背景，可以使用颜色、图片 |


#### HStack

HStack用于横向布局，子元素的对齐方式默认为center。其构造函数如下：

```swift
init(alignment: VerticalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content: () -> Content)
```

在SwiftUI中使用十分简单

#### VStack

VStack用于纵向布局，子元素的对齐方式默认为center。其构造函数如下：

```swift
init(alignment: HorizontalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content: () -> Content)
```


#### ZStack

```swift
init(alignment: Alignment = .center, @ViewBuilder content: () -> Content)
```

```swift
ZStack {
    Image("Hello")
    Text("World")
}
```

可以指定`alignment`来设置子元素的对齐方式，总共有如下的选项：

* topLeading
* top
* topTrailing
* leading
* center
* trailing
* bottomLeading
* bottom
* bottomTrailing

比如我们想让子元素底部对齐，就可以将alignment赋值为`.bottom`

```swift
ZStack(alignment: .bottom) {
    Image("Hello")
    Text("World")
}
```

### 元素

| SwiftUI | 描述 |
| ----- | ----- |
| Text | 文本 |
| Image | 图片 |
| Button | 按钮 |
| Shape | 画一个矩形 |
| Circle | 画一个圆 |

## 属性修饰器

`View`有很多属性修饰器


## `ViewBuilder`

在`ViewBuilder`提供了扩展方法可以选择性的编译内容，即在`ViewBuilder`的block中可以使用`if`、`else`语句。

```swift
static func buildEither<TrueContent, FalseContent>(first: TrueContent) -> _ConditionalContent<TrueContent, FalseContent>
static func buildEither<TrueContent, FalseContent>(second: FalseContent) -> _ConditionalContent<TrueContent, FalseContent>
static func buildIf<Content>(Content?) -> Content?
```

但是有些时候，我们需要使用`switch-case`语句来生成内容。

```swift
func contentView() -> AnyView {
    switch message.content {
    case .text(let text):
        return AnyView(TextMessageCell(text: text, isOutgoing: message.isOutgoing))
    default:
        return AnyView(EmptyView())
    }
}
```

`State` 与 `Binding`将应用的数据连接到视图上。

| 申明 | 说明 |
| -- | -- |
| `@State` | SwiftUI会管理申明为@State的属性，当属性值改变的时候，SwiftUI会重新计算body以重新绘制UI。应该只在body，或者是body调用的函数中使用。<br> 如果需要传递该值到视图树(比如body中含有一个自定义的View)，使用`$name`的形式，比如`$isPlaying` |
| `@Binding` | 使用 `@Binding` 申明视图与数据的双向连接。 |
| `@Published` | xx <br> `@Published`只能用于类中的属性，不能用于其他类型。 |
| `@ObservedObject` | xx |
| `@EnvironmentObject` | xx |


## SwiftUI 与 UIKit

苹果提供了将UIKit中的控件通过`UIViewUIViewRepresentable`和`UIViewControllerUIViewRepresentable`

```swift
import SwiftUI

struct PageControl: UIViewRepresentable {
    
    var numberOfPages: Int
    
    @Binding var currentPage: Int
    
    func makeCoordinator() -> PageControlCoordinator {
        PageControlCoordinator(self)
    }
    
    func makeUIView(context: Context) -> UIPageControl {
        let control = UIPageControl()
        control.numberOfPages = numberOfPages
        control.addTarget(self, action: #selector(PageControlCoordinator.updateCurrentPage(sender:)), for: .valueChanged)
        return control
    }
    
    func updateUIView(_ uiView: UIPageControl, context: Context) {
        uiView.currentPage = currentPage
    }
}

class PageControlCoordinator: NSObject {
    var control: PageControl
    
    init(_ control: PageControl) {
        self.control = control
    }
    
    @objc func updateCurrentPage(sender: UIPageControl) {
        control.currentPage = sender.currentPage
    }
}
```

这样在SwiftUI中就可以使用 `UIPageControl`了。

```swift
struct ChatRoomToolBoard: View {
    
    @State var currentPage = 0
    
    var body: some View {
        ZStack {
            // ...
            PageControl(numberOfPages: 2, currentPage: $currentPage)
        }
    }
}
```


## 一些不便

#### List默认Padding

List中的控件有一些默认的Padding，这对于布局来说会带来一些困扰。所以一般都是将`listRowInsets`设为空。

```swift
List(dataSource) { data in
    Row(data).listRowInsets(EdgeInsets())
}
```

#### 禁用List选中效果

可以将`buttonStyle`赋值为`PlainButtonStyle`即可达到禁用List选中效果

```swift
List(dataSource) { data in
    Row(data)
        .buttonStyle(PlainButtonStyle())
}
```

#### NavigationView 与 TabView

在页面PUSH后无法隐藏TabView

```swift
NavigationView {   
    TabView(selection: $selection){
        SessionListView()
        ContactListView()
        DiscoverListView()
        MeListView()
    }
}
```

如果使用每个Tab都使用一个NavigationView

```swift
// Content.swift
TabView(selection: $selection){
    SessionListView()
    ContactListView()
    DiscoverListView()
    MeListView()
}

// SessionListView.swift
NavigationView {   
    List {
        // ...
    }
}
```

#### NavigationLink默认箭头

使用NavigationLink进行页面的跳转，默认会显示向右的指示箭头。

```swift
NavigationLink(destination: ChatRoomView(session: session)) {
    SessionRow(session: session)
}
```

<img src="/assets/images/2020/navigationlink.jpg" width="367" />

可以使用如下的方法将指示箭头隐藏，即用一个空的视图作为导航按钮，将原视图与其用ZStack合并在一起：

```swift
ZStack {
    NavigationLink(destination: ChatRoomView(session: session)) {
        EmptyView()
    }
    .hidden()
    SessionRow(session: session)
}
```

<img src="/assets/images/2020/navigationlink_without_indicator.jpg" width="368" />

#### 不支持 NSAttributedString

解决方案：引入UILabel