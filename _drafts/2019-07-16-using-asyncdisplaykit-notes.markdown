---
layout: post
title:  "使用Texture的一些注意事项"
date:   2019-07-16 11:33:10 +0800
tag: Texture
---

### ASTableNode 不支持 separatorInset

[Issue](https://github.com/facebookarchive/AsyncDisplayKit/issues/331)

解决方案：在ASCellNode 中自己增加横线

### 给图片增加圆角

在Texture中给图片增加圆角有多种方式，官方也给出了选择哪种方式的判断理由，参考：

在应用中比较常用到的方式是：

```swift
avatarNode.cornerRadius = 6
avatarNode.cornerRoundingType = .precomposited
```


### ASImageNode tintColor

在2.8.1版本中，需要通过设置 imageModificationBlock的方式来修改 ASImageNode的tintColor

```swift
let imageNode = ASImageNode()
imageNode.imageModificationBlock = ASImageNodeTintColorModificationBlock(.white)
```


### ASTextNode passthroughNonlinkTouches

当ASTextNode 中有超链接后，点击事件默认会被屏蔽掉，即默认无法让ASTextNode中的不是超链接的其他区域响应点击事件。解决方案就是将`passthroughNonlinkTouches`属性设置为`true`。