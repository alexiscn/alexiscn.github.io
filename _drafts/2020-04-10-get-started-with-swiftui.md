---
layout: post
title:  "SwiftUI初体验"
date:   2020-04-10 09:02:10 +0800
tag: SwiftUI
---

TEST

## 生命周期

UIKit中 UIViewController有生命周期的方法，我们可以在对应的方法中处理一些特殊的逻辑。

## 布局

| SwiftUI | 描述 |UIKit |
| ---  | --- | --- |
| HStack | xxx | UIStackView |
| VStack | | UIStackView |
| ZStack | | 无 |
| Form | | |
| Section | |

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

## 一些不便

### NavigationView 与 TabView

在页面PUSH后无法隐藏TabView


### 不支持 NSAttributedString

解决方案：引入UILabel