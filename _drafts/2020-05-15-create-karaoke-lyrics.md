---
layout: post
title:  "实现卡拉OK歌词"
date:   2020-05-15 10:30:10 +0800
tag: iOS
---

需求：

* 歌词逐字播放动画
* 歌词动画能够暂停
* 歌词自动滚动
* 正在播放的行字体大小变化


分析：


```swift

class LyricsLineTableViewCell: UITableViewCell {
    func startAnimation()
    func pauseAnimation()
}

```

## Model

我们做如下的定义：

* Lyrics 代表一首歌的歌词
* Line 代表具体的某一行
* Word 代表某一行中的某个字（可能不是单一字符，比如英语单词）

那么根据歌词的定义，即：

* 歌词由很多行组成
* 行由很多字组成
* 个字有开始时间与长度
* 行也有开始时间与长度

我们能写出如下的伪代码：

```swift
struct Lyrics {
    var lines: [Line] = []
}

extension Lyrics {

    struct Line {
        var start: TimeInterval
        var duration: TimeInterval
        var words: [Word]
    }

    struct Word {
        var start: TimeInterval
        var duration: TimeInterval
        var text: String
    }
}
```

## 解析歌词文件



## 逐字动画

想要实现逐字动画，第一反应是使用`CAKeyFrameAnimation`关键帧动画

目前行业内的做法基本上类似Facebook [Shimmer](https://github.com/facebook/Shimmer)，即使用CALayer的mask属性来实现动画。

```swift
// TODO
```

## 歌词暂停与恢复

当用户暂停播放时，歌词需要停止滚动，同时动画暂停

想要使动画暂停，将其播放速度变为0，将其`timeOffset`设置为当前媒体时间。

```swift 
let animation = CAKeyFrameAnimation()

// ...
layer.add(animation, forKey: "LyricsAnimation")

func pauseAnimation() {
    let now = CACurrentMediaTime()
    let pauseTime = layer.convertTime(now, fromLayer: nil)
    animation.speed = 0
    animation.timeOffset = pauseTime
}

func resumeAnimation() {
    let pauseTime = layer.timeOffset
    layer.speed = 1
    layer.timeOffset = 0
    let npw = CACurrentMediaTime()
    let timeSincePause = layer.convertTime(now, fromLayer: nil)
    layer.beginTime = timeSincePause
}

```

## 歌词滚动

