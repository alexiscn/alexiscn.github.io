---
layout: post
title:  "mac状态栏股票应用"
date:   2015-05-05 11:33:10 +0800
tag: 开源项目
---

## 缘起

最近A股趋势不错，错过了之前的底部坑。

## NSTableView

`NSTableView` 与 `UITableView` 有相似的地方，也有很多不同的地方。


#### 滑动

NSTableView 本身不支持滑动，需要嵌套在 NSScrollView 中才能滑动。


#### 分隔符

## 


### 纯代码 NSViewController

默认NSViewController 不能由代码实例化，如果 let controller = YourNSViewController 会报如下的错误信息

```swift
-[NSNib _initWithNibNamed:bundle:options:] could not load the nibName: ProjectName.YourNSViewController in bundle (null).
```

借助如下的代码

```swift
override func loadView() {
    self.view = NSView()
}
```

### 修改视图的背景颜色

不同于 UIView， NSView 并不提供 backgroundColor属性来修改自身的背景颜色

```swift
view.wantsLayer = true
view.layer.backgroundColor = NSColor.white.cgColor
```