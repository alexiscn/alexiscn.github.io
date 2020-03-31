---
layout: post
title:  "iOS 视频转场效果"
date:   2020-03-08 19:02:10 +0800
tag: AVFoundation
---

![](/assets/images/2020/video-transitions.gif)

## 需求分析

我们来简单分析一下合并视频添加转场效果的需求，用输入和输出来分解。

> 输入值

- 一组视频，在AVFoundation中可以用 AVAsset 表示一个视频
- 指定的一个转场效果
- 转场的时间长度，即用于转场的时间长度，一般在第一段视频的结尾和第二段视频的开头。

> 输出结果

- 这些视频合并为一个视频，并在衔接处有转场动画
- 如果有错误，抛出错误信息

用伪代码表示，即希望提供一个这样的类，暴露出来一个合并视频的方法。

```swift
class TransitionManager {

    func merge(videos: [AVAsset], 
               with effect: TransitionEffect, 
               transitionDuration: CMTimeRange, 
               completion: @escaping (VideoResult) -> Void) {
        // do work
    }
}
```


在了解如何合并之前，我们先来熟悉一下常用的几个数据结构

## CMTime

`CMTime`是一个表示时间值（比如时间戳或者时长）的结构体，定义如下：


```swift
public struct CMTime {

    public var value: CMTimeValue

    public var timescale: CMTimeScale

    public var flags: CMTimeFlags 

    public var epoch: CMTimeEpoch

}
```

`timescale` 。比如`timescale`为4，则每个单元表示1/4秒，如果`timescale`为10，则每个单元表示0.1秒。

记住以下的公式就能很好的理解 CMTime

```swift
second = value / timescale
```

### 常用的操作


## CMTimeRange

`CMTimeRange`用来表示一个时间区间，他有两个属性，开始时间`start`，时间长度`duration`。


## 开始合成

苹果在 iOS7 中增加了视频合成处理的功能 `AVVideoCompositing`

```swift
public protocol AVVideoCompositing : NSObjectProtocol {

    var sourcePixelBufferAttributes: [String : Any]? { get }

    var requiredPixelBufferAttributesForRenderContext: [String : Any] { get }

    func renderContextChanged(_ newRenderContext: AVVideoCompositionRenderContext)

    func startRequest(_ asyncVideoCompositionRequest: AVAsynchronousVideoCompositionRequest)
}
```

#### 合成的步骤

- 创建 AVMutableComposition
- 往 AVMutableComposition 添加 videoTrack 和 audioTrack
- xxxx





#### 简单的视频拼接




## 视频导出

`AVAssetExportSession`


开源地址：[MTTransitions](https://github.com/alexiscn/MTTransitions)


## 遇到的问题


1. 转场效果可以播放但导出失败

```swift
Error Domain=AVFoundationErrorDomain Code=-11838 "Operation Stopped" UserInfo={NSLocalizedFailureReason=The operation is not supported for this media., NSLocalizedDescription=Operation Stopped, NSUnderlyingError=0x2808acde0 {Error Domain=NSOSStatusErrorDomain Code=-16976 "(null)"}}
```

导致错误的原因是，进行合并的视频没有音频文件，但用来导出的 AVMutableComposition 却添加了 音频文件的 Track.


参考链接

* [CMTime](https://developer.apple.com/documentation/coremedia/cmtime-u58)
* [CMTimeRange]()
* [AVCustomEdit](https://developer.apple.com/library/archive/samplecode/AVCustomEdit/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013411-Intro-DontLinkElementID_2)