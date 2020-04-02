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

苹果推荐`CVPixelBuffer`用`AVAssetWriterInputPixelBufferAdaptor`的[pixelBufferPool来创建以提高性能，参考 [AVAssetWriterInputPixelBufferAdaptor](https://developer.apple.com/documentation/avfoundation/avassetwriterinputpixelbufferadaptor)。

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