---
layout: post
title:  "iOS图片创建视频"
date:   2020-03-31 10:02:10 +0800
tag: AVFoundation
---

大学毕业的时候，做过班级的相册，就是把班上所有的人照片用一个相册软件做成一个mp4文件。在iOS中也可以把一组图片做成一个视频，还可以用[MTTransitions](https://github.com/alexiscn/MTTransitions)增加图片转成效果。


## AVAssetWriter

`AVAssetWriter`可以将音视频媒体数据写入文件。在录制视频的时候经常会用到。其基本的使用方法如下：

* init(url:fileType) 或者 init(outputURL:fileType) 来创建一个`AVAssetWriter`对象
* 往assetWriter中添加`AVAssetWriterInput`对象
* startSession(atSourceTime:)开始写入会话
* 将媒体数据通过 `AVAssetWriterInput`对象写入文件
* finishWriting 来标记完成写入 

更多的可以参考苹果的示例 [RosyWriter/MovieRecorder.m](https://developer.apple.com/library/archive/samplecode/RosyWriter/Listings/Classes_Utilities_MovieRecorder_m.html#//apple_ref/doc/uid/DTS40011110-Classes_Utilities_MovieRecorder_m-DontLinkElementID_23)

## AVAssetWriterInput

`AVAssetWriterInput`是`AVAssetWriter`的输入，可以添加音频、视频输入

```swift
let videoSettings: [String: Any] = [
            AVVideoCodecKey: AVVideoCodecH264,
            AVVideoWidthKey: outputSize.width,
            AVVideoHeightKey: outputSize.height
        ]
let writerInput = AVAssetWriterInput(mediaType: .video, outputSettings: videoSettings)
```

需要指定`outputSettings`属性，是一个字典。

其中视频可以用的Key可以在`AVVideoSettings.h`中找到.
音频可用的Key可以在`AVAudioSettings.h`，位于`AVFoundation/AVFAudio`下。

## AVAssetWriterInputPixelBufferAdaptor

有了`AVAssetWriterInput`后，需要通过`AVAssetWriterInputPixelBufferAdaptor`将视频作为`CVPixelBuffer`写入到`AVAssetWriterInput`中，需要指定PixelBuffer的信息：

```swift
let attributes: [String: Any] = [
            (kCVPixelBufferPixelFormatTypeKey as String): kCVPixelFormatType_32BGRA,
            (kCVPixelBufferWidthKey as String): outputSize.width,
            (kCVPixelBufferHeightKey as String): outputSize.height
        ]

let pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(assetWriterInput: writerInput
                         sourcePixelBufferAttributes: attributes)
pixelBufferAdaptor.append(buffer, withPresentationTime: presentTime)
```

苹果推荐`CVPixelBuffer`用`AVAssetWriterInputPixelBufferAdaptor`的`pixelBufferPool`来创建以提高性能，参考 [AVAssetWriterInputPixelBufferAdaptor](https://developer.apple.com/documentation/avfoundation/avassetwriterinputpixelbufferadaptor)。

```swift
guard let pixelBufferPool = pixelBufferAdaptor.pixelBufferPool else {
    fatalError("AVAssetWriterInputPixelBufferAdaptor pixelBufferPool empty")
}
var pixelBuffer: CVPixelBuffer?
CVPixelBufferPoolCreatePixelBuffer(kCFAllocatorDefault, pixelBufferPool, &pixelBuffer)
```


## 图片转场

对于图片转场比较简单，就是模拟progress从0到1，然后将其输出写入文件。这里对于每个转场写入了30帧图像，计算每帧对应的progress与presentTime。

```swift
guard let pixelBufferPool = pixelBufferAdaptor.pixelBufferPool else {
    fatalError("AVAssetWriterInputPixelBufferAdaptor pixelBufferPool empty")
}

// ...

var index = 0
while index < (images.count - 1) {
    var presentTime = CMTimeMake(value: Int64(frameDuration * Double(index) * 1000), timescale: 1000)
    let transition = effects[index].transition
    transition.inputImage = images[index]
    transition.destImage = images[index + 1]
    transition.duration = transitionDuration
    
    let frameBeginTime = presentTime
    let frameCount = 29
    for counter in 0 ... frameCount {
        autoreleasepool {
            while !writerInput.isReadyForMoreMediaData {
                Thread.sleep(forTimeInterval: 0.01)
            }
            let progress = Float(counter) / Float(frameCount)
            transition.progress = progress
            let frameTime = CMTimeMake(value: Int64(transitionDuration * Double(progress) * 1000), timescale: 1000)
            presentTime = CMTimeAdd(frameBeginTime, frameTime)
            var pixelBuffer: CVPixelBuffer?
            CVPixelBufferPoolCreatePixelBuffer(kCFAllocatorDefault, pixelBufferPool, &pixelBuffer)
            if let buffer = pixelBuffer, let frame = transition.outputImage {
                try? MTTransition.context?.render(frame, to: buffer)
                pixelBufferAdaptor.append(buffer, withPresentationTime: presentTime)
            }
        }
    }
    index += 1
}
```

## 增加背景音乐

增加背景音乐其实就是将音频文件与视频文件进行混合，我们这里创建一个`AVMutableComposition`，然后将音频与视频添加进合成器，最后使用`AVAssetExportSession`进行导出。

为了简单起见，我们假定整段视频都增加音乐。当然也可以设置音乐的TimeRange，即只在某个时间段增加音乐。

```swift
private func mixAudio(_ audio: AVAsset, 
                      video: AVAsset, 
                      completion: MTMovieMakerCompletion? = nil) throws {
    guard let videoTrack = video.tracks(withMediaType: .video).first else {
        fatalError("Can not found videoTrack in Video File")
    }
    guard let audioTrack = audio.tracks(withMediaType: .audio).first else {
        fatalError("Can not found audioTrack in Audio File")
    }
    
    let composition = AVMutableComposition()
    guard let videoComposition = composition.addMutableTrack(withMediaType: .video, preferredTrackID: CMPersistentTrackID(1)),
        let audioComposition = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: CMPersistentTrackID(2)) else {
        return
    }
    // TODO:
}
```

由于是给视频文件添加背景音乐，所以最后生成的视频文件长度肯定等于输入视频的长度。将视频添加到合成器中十分简单：

```swift
let videoTimeRange = CMTimeRange(start: .zero, duration: video.duration)
try videoComposition.insertTimeRange(videoTimeRange, of: videoTrack, at: .zero)
```

将音频插入合成器中需要考虑两种场景：

* 音频文件长度大于等于视频文件长度
* 音频文件长度小于视频长度

对于音频长度大于视频长度的，处理起来就十分简单，直接将音频截取视频长度，从头插入到 AVMutableComposition中就可以了。

![](/assets/images/2020/AVComposition02.jpg)

```swift
let audioTimeRange = CMTimeRangeMake(start: .zero, duration: video.duration)
try audioComposition.insertTimeRange(audioTimeRange, of: audioTrack, at: .zero)
```

而对于音频长度小于视频长度的，我们需要重复将音频插入到AVMutableComposition中，需要计算每个音频段的TimeRange。

![](/assets/images/2020/AVComposition01.jpg)

```swift
let repeatCount = Int(video.duration.seconds / audio.duration.seconds)
let remain = video.duration.seconds.truncatingRemainder(dividingBy: audio.duration.seconds)
let audioTimeRange = CMTimeRange(start: .zero, duration: audio.duration)
for i in 0 ..< repeatCount {
    let start = CMTime(seconds: Double(i) * audio.duration.seconds, preferredTimescale: audio.duration.timescale)
    try audioComposition.insertTimeRange(audioTimeRange, of: audioTrack, at: start)
}
if remain > 0 {
    let startSeconds = Double(repeatCount) * audio.duration.seconds
    let start = CMTime(seconds: startSeconds, preferredTimescale: audio.duration.timescale)
    let remainDuration = CMTime(seconds: remain, preferredTimescale: audio.duration.timescale)
    let remainTimeRange = CMTimeRange(start: .zero, duration: remainDuration)
    try audioComposition.insertTimeRange(remainTimeRange, of: audioTrack, at: start)
}
```

最后使用`AVAssetExportSession`导出即可，这里就不贴代码了。


## 注意点

writerInput必须要在`isReadyForMoreMediaData`为`true`的时候，才能正常写入数据，否则会抛出异常。所以我们在频繁写入数据之前，要确保改值为`true`，否则需要让线程等待writerInput处理完上一个数据。

```swift
while !writerInput.isReadyForMoreMediaData {
    Thread.sleep(forTimeInterval: 0.01)
}
```

## 内存暴涨

在给[MTTransitions](https://github.com/alexiscn/MTTransitions) 中的每个转场效果生成预览视频的时候，由于不经意的bug（手动狗头），发现`MTMovieMaker`会遇到内存疯涨的情况。先说一下这个bug：

我把转场过程中的所有图片都存下来，可能有120张左右。然后对这120张图片去创建视频。按照上面的写写法，总共有119次外层循环（即对每两张照片进行转场），内层循环有30次。相当于对于每次图片转场都需要处理 120x30 = 3600 张图片，假如一张图 0.3M，一下子1.08G内存没有了...

实际上只需要传入两张照片即可，对这两张照片进行转场计算。不过正是由于这个bug发现了潜在的问题

```swift
autoreleasepool {
    // 大量内存操作
}
```

如上代码都能在 [MTMovieWriter](https://github.com/alexiscn/MTTransitions/blob/master/Source/MTMovieMaker.swift)中找到。