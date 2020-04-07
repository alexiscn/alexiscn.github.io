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

记住以下的公式就能很好的理解 CMTime，value除以timescale就是我们平常理解的秒。

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

public func + (lhs: CMTime, rhs: CMTime) -> CMTime {
    return CMTimeAdd(lhs, rhs)
}

public func - (lhs: CMTime, rhs: CMTime) -> CMTime {
    return CMTimeSubtract(lhs, rhs)
}
```

这样下面的两行代码是等效的

```swift
let time3 = CMTimeAdd(time1, time2)

let time3 = time1 + time2
```


## CMTimeRange

`CMTimeRange`用来表示一个时间区间，他有两个属性，开始时间`start`，时间长度`duration`，结束时间可以通过开始时间与时间长度计算得到。通常使用 `CMTimeRangeMake(start: CMTime, duration: CMTime) -> CMTimeRange` 来创建一个`CMTimeRange`对象。`CMTimeRange`在合成操作的时候会经常用到，比如在计算应该在哪个时间段做转场效果，应该将视频插入到哪个时间段等等。

```swift
typedef struct {
    CMTime start;
    CMTime duration;
} CMTimeRange;
```




## 合成

苹果在 iOS7 中增加了视频合成处理的功能 `AVVideoCompositing`。对于`AVPlayerItem`, `AVAssetExportSession`, `AVAssetImageGenerator`, `AVAssetReaderVideoCompositionOutput`实例，如果其videoComposition不为nil，系统会适用自定义的合成器。

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


#### 简单的视频拼接

简单的视频拼接是指将视频直接合并，没有转场效果。按照一般的理解就是简单的把两个视频合并在一起。按照上述的合成步骤，其伪代码实现大致如下：

![](/assets/images/2020/composition_normal@2x.jpg)

#### 带有转场效果的拼接

带有转场效果的拼接，就是把上一段视频的末尾和下一段视频的开头部分做一些像素操作。比如渐变的转场效果，就是把A的alpha从1变为0，B的alpha从0变为1，然后做[mix](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/mix.xhtml)运算。

![](/assets/images/2020/composition_transition@2x.jpg)



## 视频导出

我们需要使用`AVAssetExportSession`将合成的视频导出到本地，首先需要使用`AVAssetExportSession(asset:presetName:)`来初始化`AVAssetExportSession`对象，然后配置输出属性，如`videoComposition`、输出的文件格式、输出的文件路径、输出的时间段等等。

`presetName`是一个字符串，有多种选择

| 取值 | 说明 | 备注 |
| --- | --- | --- |
| AVAssetExportPresetLowQuality | 导出低质量电影(视频H264压缩，音频AAC压缩) | iOS4.0 + |
| AVAssetExportPresetMediumQuality | 导出中等质量电影(视频H264压缩，音频AAC压缩) | iOS4.0 + |
| AVAssetExportPresetHighestQuality | 导出最高质量电影(视频H264压缩，音频AAC压缩) |iOS4.0 + |
| AVAssetExportPresetHEVCHighestQuality | 导出HEVC格式电影(视频HEVC压缩，音频AAC压缩)  | iOS11.0 + |
| AVAssetExportPresetHEVCHighestQualityWithAlpha | 导出带Alpha通道的HEVC格式电影(视频HEVC压缩，音频AAC压缩) | iOS13.0 + |
| AVAssetExportPreset640x480 | 导出分辨率为640x480的电影 | iOS4.0 + |
| AVAssetExportPreset960x540 | 导出分辨率为960x540的电影 | iOS4.0 + |
| AVAssetExportPreset1280x720 | 导出分辨率为1280x720的电影 | iOS4.0 + |
| AVAssetExportPreset1920x1080 | 导出分辨率为1920x1080的电影 | iOS5.0 + |
| AVAssetExportPreset3840x2160 | 导出分辨率为3840x2160的电影 | iOS9.0 + |
| AVAssetExportPresetHEVC1920x1080 | 导出分辨率为1920x1080HECV格式的电影 | iOS11.0 + |
| AVAssetExportPresetHEVC1920x1080WithAlpha | 导出分辨率为1920x1080带Alpha通道HECV格式的电影 | iOS13.0 + |
| AVAssetExportPresetHEVC3840x2160 | 导出分辨率为3840x2160HECV格式的电影 | iOS11.0 + |
| AVAssetExportPresetHEVC3840x2160WithAlpha | 导出分辨率为3840x2160带Alpha通道HECV格式的电影 | iOS13.0 + |
| AVAssetExportPresetAppleM4A | 只导出 .m4a 音频文件 | iOS4.0 + |
| AVAssetExportPresetPassthrough | Passthrough | iOS4.0 +| 

详细的可以参考 

* [Export Preset Names for Device-Appropriate QuickTime Files](https://developer.apple.com/documentation/avfoundation/avassetexportsession/export_preset_names_for_device-appropriate_quicktime_files)
* [Export Preset Names for QuickTime Files of a Given Size](https://developer.apple.com/documentation/avfoundation/avassetexportsession/export_preset_names_for_quicktime_files_of_a_given_size)

我们可以封装一个导出`AVComposition`的类，传入之前得到的结果。`AVAssetExportSession`提供了一个`progress`的属性来查询当前的导出进度，该值并不是`key-value observable`。所以我们无法用`key-value`去得到进度变化。 我们可以写一个Timer然后去查询当前的导出进度。

```swift
import AVFoundation

public typealias MTVideoExporterCompletion = (Error?) -> Void

public class MTVideoExporter {
    
    private let composition: AVMutableComposition
    
    private let videoComposition: AVMutableVideoComposition
    
    private let exportSession: AVAssetExportSession
    
    public convenience init(transitionResult: MTVideoTransitionResult, presetName: String = AVAssetExportPresetHighestQuality) throws {
        try self.init(composition: transitionResult.composition, videoComposition: transitionResult.videoComposition, presetName: presetName)
    }
    
    public init(composition: AVMutableComposition,
                videoComposition: AVMutableVideoComposition,
                presetName: String = AVAssetExportPresetHighestQuality) throws {
        self.composition = composition
        self.videoComposition = videoComposition
        guard let session = AVAssetExportSession(asset: composition, presetName: presetName) else {
            fatalError("Can not create AVAssetExportSession, please check composition")
        }
        self.exportSession = session
        self.exportSession.videoComposition = videoComposition
    }
    
    public func export(to fileURL: URL, outputFileType: AVFileType = .mp4, completion: @escaping MTVideoExporterCompletion) {
        
        let fileExists = FileManager.default.fileExists(atPath: fileURL.path)
        if fileExists {
            do {
                try FileManager.default.removeItem(atPath: fileURL.path)
            } catch {
                print("An error occured deleting the file: \(error)")
            }
        }
        exportSession.outputURL = fileURL
        exportSession.outputFileType = outputFileType
        
        let startTime = CMTimeMake(value: 0, timescale: 1)
        let timeRange = CMTimeRangeMake(start: startTime, duration: composition.duration)
        exportSession.timeRange = timeRange
        
        exportSession.exportAsynchronously(completionHandler: { [weak self] in
            completion(self?.exportSession.error)
        })
    }
}
```

## 遇到的问题


1. 转场效果可以播放但导出失败

```swift
Error Domain=AVFoundationErrorDomain Code=-11838 "Operation Stopped" UserInfo={NSLocalizedFailureReason=The operation is not supported for this media., NSLocalizedDescription=Operation Stopped, NSUnderlyingError=0x2808acde0 {Error Domain=NSOSStatusErrorDomain Code=-16976 "(null)"}}
```

导致错误的原因是，进行合并的视频没有音频文件，但用来导出的 AVMutableComposition 却添加了 音频文件的 Track.

开源地址：[MTTransitions](https://github.com/alexiscn/MTTransitions)

## 参考链接

* [CMTime](https://developer.apple.com/documentation/coremedia/cmtime-u58)
* [CMTimeRange]()
* [AVCustomEdit](https://developer.apple.com/library/archive/samplecode/AVCustomEdit/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013411-Intro-DontLinkElementID_2)