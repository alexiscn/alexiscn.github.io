---
layout: post
title:  "火币网Mac小应用"
date:   2018-02-01 17:42:10 +0800
tag: 开源项目
---

最近在看比特币相关的知识，也入了一点比特币（0.1个）... 然后发现[火币网](https://www.huobipro.com) 提供了 API


用到的类库

* Starcream， 用于WebSocket
* GzipSwift, 用于gizp解压缩socket收到的数据包
* GenericNetworking，用于网络请求


代码已开源在 [HuobiMac](https://github.com/alexiscn/huobimac)

1、将应用设置为状态栏应用

状态栏应用即只是在状态栏上出现应用图标，不出现主窗口，在点击应用图标时弹出浮窗，显示程序的主界面。默认创建的Cocoa App会有一个主窗口

![](/assets/images/cocoa-app-default-window.jpg)

* 打开Main.storyboard，删除 Window Controller Scene
* 在Info.plist 中增加 Application is agent(UIElement)，值设置为YES
* 设置状态栏图标以及处理点击图标事件

```swift
class AppDelegate: NSObject, NSApplicationDelegate {
    let statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)

    ...

    func applicationDidFinishLaunching(_ aNotification: Notification) {
        if let button = statusItem.button {
            button.image = NSImage(named: NSImage.Name(rawValue: "StatusBarButtonImage"))
            button.imagePosition = .imageLeft
        }
    }
}
```


2、处理点击状态栏图标事件

在默认状态下，点击状态栏图标是显示浮窗展示状态栏应用的界面，再点击状态栏图标应该是将浮窗隐藏。

```swift

class AppDelegate: NSObject, NSApplicationDelegate {
    let statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)

    let popover = NSPopover()
}

```

在Cocoa应用中，可以使用`NSPopover`来展示浮窗内容，主要设置 `NSPopover`的contentViewController 属性来设置需要展示的ViewController。

```swift
func applicationDidFinishLaunching(_ aNotification: Notification) {

    let mainViewController = MainViewController()
    popover.contentViewController = mainViewController
    popover.appearance = NSAppearance(named: .aqua)
    popover.animates = false
    popover.behavior = .transient
}
```