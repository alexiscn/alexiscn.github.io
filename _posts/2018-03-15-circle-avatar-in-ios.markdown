---
layout: post
title:  "iOS圆角实现方式"
date:   2018-03-15 11:22:10 +0800
tag: iOS杂货铺
---

在iOS中实现圆角有多种方式，最简单的就是设置layer.cornerRadius属性

![](/assets/images/circleavatar.jpg)

方式一：layer.cornerRadius

```swift
let avatarButton = UIButton(type: .custom)
avatarButton.frame = CGRect(x: 100, y: 20, width: 90, height: 90)
avatarButton.setImage(UIImage(named: "avatar.jpg"), for: .normal)
avatarButton.layer.cornerRadius = avatarButton.frame.size.width/2
avatarButton.clipsToBounds = true

container.addSubview(avatarButton)
```

方式二：利用CoreGraphic进行裁切

```swift
var avatarImageView = UIImageView()
avatarImageView.frame = CGRect(x: 100, y: 120, width: 90, height: 90)

var image = UIImage(named: "avatar.jpg")
UIGraphicsBeginImageContextWithOptions(avatarImageView.bounds.size, false, 0.0)
UIBezierPath(roundedRect: avatarImageView.bounds, cornerRadius: avatarImageView.bounds.width/2.0).addClip()
image?.draw(in: avatarImageView.bounds)
image = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()
avatarImageView.image = image
container.addSubview(avatarImageView)
```


方式三：视图叠加

这种方式有点tricky，实际上并没有对原来的视图做圆角处理，在上面加了一个圆角的图片，中心圆部分是透明的，外面是背景颜色（如白色）


![](/assets/images/corner_circle.png)


```swift
let avatarButton3 = UIButton(type: .custom)
avatarButton3.frame = CGRect(x: 100, y: 220, width: 90, height: 90)
avatarButton3.setImage(UIImage(named: "avatar.jpg"), for: .normal)

let imageView = UIImageView(frame: CGRect(x: 0, y: 0, width: 100, height: 100))
imageView.image = UIImage(named: "corner_circle")
imageView.center = avatarButton3.center
container.addSubview(avatarButton3)
container.addSubview(imageView)
```

如果项目比较简单并且对性能要求不是特别高，推荐使用第一种方式，简洁方便。如果在滑动列表，要求对头像进行圆角处理，推荐使用第三种方式，性能较高，不需要进行计算圆角与裁切。


以上的代码都能在 [Playgrounds/CircleAvatar](https://github.com/alexiscn/playgrounds) 找到