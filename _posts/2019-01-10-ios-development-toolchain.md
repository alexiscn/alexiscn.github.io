---
layout: post
title:  "iOS开发工具链"
date:   2019-01-10 10:03:15 +0800
tag: iOS杂货铺
---

> 工欲善其事，必先利其器

## Xcode

iOS开发当然需要用Xcode了，Xcode相对前优化的比以前的版本好很多，但是对Swift的智能提示不是十分友好。

## 图片处理

iOS开发中或多或少会跟图片打交道，比如图标的压缩。

#### TinyPng

[TinyPng](https://tinypng.com/)是非常棒的在线压缩服务，支持png、jpeg格式。 有一定的限制，单张图片的大小不能超过5M，一次不超过20张

![TinyPng](/assets/images/2019/ios_tinypng.jpg)

#### ImageOptim

[ImageOptim](https://imageoptim.com/mac) 是一款mac上运行的压缩工具。运行效率效率稍微不如TinyPng那么快，但是支持5M以上的图片。可以作为TinyPng的辅助。

![ImageOptim](/assets/images/2019/ImageOptim-app.png)

#### Preview

没错，就是系统自带的预览软件。可以用来修改图片的分辨率，十分的方便。

##### IconFont

[iconfont](https://www.iconfont.cn) 是阿里巴巴开源的矢量图标库。项目中的一些图标可以

![IconFont](/assets/images/2019/ios_iconfont.jpg)

## 格式化

#### JSON格式化

在线格式化工具有很多，kjson比较方便，域名也好记忆。

![KJSON](/assets/images/2019/ios_kjson.jpg)

## 网络

网络抓包 Charles