---
layout: post
title:  "Cocoa项目：StocksBar"
date:   2019-05-05 11:33:10 +0800
tag: 开源项目
---

## 缘起

最近A股趋势不错，错过了之前的底部坑。于是就想着做一个Mac状态栏应用，可以在工作的时候偶尔关注一下股票的行情。

## 股票API

股票API用的新浪股票的接口，相对来说比较简单

http://hq.sinajs.cn/list=sh601933

返回结果如下：

```js
var hq_str_sh601933="永辉超市,9.390,9.480,9.400,9.650,9.310,9.400,9.420,45031643,427349791.000,38767,9.400,19100,9.390,26500,9.380,14000,9.370,20400,9.360,1200,9.420,9300,9.430,39200,9.440,122072,9.450,26300,9.460,2019-05-09,15:00:00,00";
```

而且还能一次请求多个股票的数据，用逗号分隔即可。

### 行情

### 搜索建议



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