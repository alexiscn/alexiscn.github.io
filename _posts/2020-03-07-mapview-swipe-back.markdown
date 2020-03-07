---
layout: post
title:  "iOS Tips: 解决MKMapView无法响应interactivePopGestureRecognizer"
date:   2020-03-07 12:01:10 +0800
tag: tips
---

假如页面显示了地图控件，由于地图本身可以拖动来查看其他区域，其内置的 `UIPanGestureGestureRecognizer` 与系统导航栏的返回手势冲突了，导致无法右滑屏幕边缘退出页面。

![](/assets/images/2020/mapview-fix-interactive-pop-gesture@2x.jpeg)

有一个很简单的解决方案，即在页面中增加一个不可见的视图覆盖在地图视图上方，当手指滑动到该区域的时候，由于地图被该视图覆盖在上面，所以无法响应地图的拖动事件。

```swift
private func fixNavigationSwipeGesture() {
    let transparent = UIView()
    transparent.backgroundColor = .clear
    transparent.frame = CGRect(x: 0, y: 0, width: 16, height: view.bounds.height)
    view.addSubview(transparent)
}
```