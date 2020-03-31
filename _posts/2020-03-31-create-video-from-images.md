---
layout: post
title:  "iOS图片创建视频"
date:   2020-03-31 10:02:10 +0800
tag: AVFoundation
---

大学毕业的时候，做过班级的相册，就是把班上所有的人


## AVAssetWriter


## AVAssetWriterInput


## 内存暴涨

在给[MTTransitions](https://github.com/alexiscn/MTTransitions) 中的每个转场效果生成预览视频的时候，由于不经意的bug（手动狗头），发现`MTMovieMaker`会遇到内存疯涨的情况。先说一下这个bug：

我把转场过程中的所有图片都存下来，可能有120张左右。然后对这120张图片去创建视频。按照上面的写写法，总共有119次外层循环（即对每两张照片进行转场），对

实际上只需要传入两张照片即可，对这两张照片进行转场计算。不过正是由于这个bug发现了潜在的问题

```swift
autoreleasepool {
    // 大量内存操作
}
```