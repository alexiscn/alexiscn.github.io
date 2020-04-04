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

#### 常用的操作

CoreMedia提供了一些全局方法来操作CMTime对象

| 方法 | 说明 |
| -- | -- |
| CMTimeAdd(CMTime, CMTime) -> CMTime | 两个时间相加 |
| CMTimeSubtract(CMTime, CMTime) -> CMTime | 两个时间相减 |
| CMTimeMultiply(CMTime, multiplier: Int32) -> CMTime | 两个时间相乘 |
| CMTimeGetSeconds(CMTime) -> Float64 | 将CMTime转为秒 |
| CMTimeCompare(CMTime, CMTime) -> Int32 | 比较两个CMTime对象 |
| CMTimeMaximum(CMTime, CMTime) -> CMTime | 获取两个CMTime中比最大的 |
| CMTimeMinimum(CMTime, CMTime) -> CMTime | 获取两个CMTime中比最小的 |
| CMTIME_IS_VALID(CMTime) -> Bool | 检查CMTime是否有效 |
| CMTIME_IS_INVALID(CMTime) -> Bool | 检查CMTime是否无效 |

如果需要经常对CMTime进行运算，比如时间相加，我们可以重载操作符来简化代码：

```swift
import CoreMedia

public func +=( lhs: inout CMTime, rhs: CMTime) {
    lhs = lhs + rhs
}

public func -=( lhs: inout CMTime, rhs: CMTime) {
    lhs = lhs - rhs
}
```

这样下面的两行代码是等效的

```swift
let time3 = CMTimeAdd(time1, time2)

let time3 = time1 + time2
```


## CMTimeRange

`CMTimeRange`用来表示一个时间区间，他有两个属性，开始时间`start`，时间长度`duration`，结束时间可以通过开始时间与时间长度计算得到。通常使用 `CMTimeRangeMake(start: CMTime, duration: CMTime) -> CMTimeRange` 来创建一个`CMTimeRange`对象。`CMTimeRange`在合成操作的时候会经常用到。


## 合成

苹果在 iOS7 中增加了视频合成处理的功能 `AVVideoCompositing`

```swift
public protocol AVVideoCompositing : NSObjectProtocol {

    var sourcePixelBufferAttributes: [String : Any]? { get }

    var requiredPixelBufferAttributesForRenderContext: [String : Any] { get }

    func renderContextChanged(_ newRenderContext: AVVideoCompositionRenderContext)

    func startRequest(_ asyncVideoCompositionRequest: AVAsynchronousVideoCompositionRequest)
}
```

### 合成的步骤

- 创建 AVMutableComposition
- 往 AVMutableComposition 添加 videoTrack 和 audioTrack
- 计算每个Track对应的TimeRange
- 进行合成与导出


```swift
let composition = AVMutableComposition()
composition.naturalSize = videoSize
```

```swift
let videoComposition = AVMutableVideoComposition()
videoComposition.customVideoCompositorClass = MTVideoCompositor.self
videoComposition.frameDuration = CMTimeMake(value: 1, timescale: 30) // 30 fps.
videoComposition.renderSize = videoSize
```


### 简单的视频拼接

简单的视频拼接是指将视频直接合并，没有转场效果。按照一般的理解就是简单的把两个视频合并在一起。按照上述的合成步骤，其伪代码实现大致如下：

![](/assets/images/2020/composition_normal@2x.jpg)

#### 带有转场效果的拼接

带有转场效果的拼接，就是把上一段视频的末尾和下一段视频的开头部分做一些像素操作

![](/assets/images/2020/composition_transition@2x.jpg)

## 视频导出

`AVAssetExportSession`


开源地址：[MTTransitions](https://github.com/alexiscn/MTTransitions)


## 遇到的问题


1. 转场效果可以播放但导出失败

```swift
Error Domain=AVFoundationErrorDomain Code=-11838 "Operation Stopped" UserInfo={NSLocalizedFailureReason=The operation is not supported for this media., NSLocalizedDescription=Operation Stopped, NSUnderlyingError=0x2808acde0 {Error Domain=NSOSStatusErrorDomain Code=-16976 "(null)"}}
```

导致错误的原因是，进行合并的视频没有音频文件，但用来导出的 AVMutableComposition 却添加了 音频文件的 Track.


## 参考链接

* [CMTime](https://developer.apple.com/documentation/coremedia/cmtime-u58)
* [CMTimeRange]()
* [AVCustomEdit](https://developer.apple.com/library/archive/samplecode/AVCustomEdit/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013411-Intro-DontLinkElementID_2)