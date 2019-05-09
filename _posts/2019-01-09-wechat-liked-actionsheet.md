---
layout: post
title:  "仿微信ActionSheet"
date:   2019-01-09 18:03:15 +0800
tag: 开源项目
---

[V2EX项目](https://github.com/alexiscn/v2ex)中使用了一个类似[微信的弹窗](https://github.com/alexiscn/WXActionSheet)，抽时间将其重构了下做了一个小[开源库](https://github.com/alexiscn/WXActionSheet)。

![](/assets/images/2019/wxactionsheet_preview.gif)


用法也很简单，类似`UIAlertController`一样，添加具体的Action，然后直接show就行了。更多用法可以参考 [Sample](https://github.com/alexiscn/WXActionSheet)


```swift

let actionSheet = WXActionSheet(cancelButtonTitle: "取消")
actionSheet.add(WXActionSheetItem(title: "发送给朋友", handler: { _ in
    
}))
actionSheet.add(WXActionSheetItem(title: "收藏", handler: { _ in
    
}))
actionSheet.add(WXActionSheetItem(title: "保存图片", handler: { _ in
    
}))
actionSheet.add(WXActionSheetItem(title: "删除", handler: { _ in
    
}, type: .destructive))
actionSheet.show()

```

原理也相对比较简单，看代码就能懂。遇到的一些坑是将这个View添加到哪个层级上。一开始使用的是 `keyWindow`，发现当页面显示有异常。然后使用的是 `UIApplication.shared.windows.last`，发现当页面上有`WKWebView`时，聚焦WebView后，弹窗就显示不出来。利用Xcode的截图功能看了下，发现最上层的Window是`UIRemoteKeyboardWindow`。

最后使用的是

```swift
 let windows = UIApplication.shared.windows.filter { NSStringFromClass($0.classForCoder) != "UIRemoteKeyboardWindow" }
guard let win = windows.last else { return }
```

找到不是`UIRemoteKeyboardWindow`的最上层的一个`window`。