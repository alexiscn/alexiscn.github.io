---
layout: post
title:  "实现卡拉OK歌词"
date:   2020-05-15 10:30:10 +0800
tag: iOS
---

## 需求分析：

一个基本的卡拉OK歌词App（如QQ音乐歌词、全民K歌）具备如下的功能：

* 歌词逐字播放动画
* 歌词动画能够暂停
* 歌词自动滚动
* 正在播放的歌词字体大小变化


分析：


```swift

class LyricsTableViewCell: UITableViewCell {
    func startAnimation()
    func pauseAnimation()
}

```

## Model设计

根据一般的歌词设计，基本都是每一句歌词都有开始时间及时长。每句歌词中的每个字也有开始时间与长度，这样就能对每个字进行动画播放。我们以QQ音乐歌词为例：


```
[ti:屋顶]
[ar:周杰伦]
[al:爱回温 (新歌+精选)]
[by:]
[offset:0]
[10,10]屋顶 - 温岚 (Landy Wen)/周杰伦 (Jay Chou)(10,10)
[20,20]词：周杰伦(20,20)
[30,30]曲：周杰伦(30,30)
[40,40]编曲：屠颖(40,40)
[50,50]制作人：周杰伦(50,50)
[24092,4969]男：半(24092,250)夜(24342,280)睡(24622,350)不(24972,300)着(25272,320)觉 (25592,650)把(26682,310)心(26992,330)情(27322,340)哼(27662,310)成(27972,320)歌(28292,769)
[29431,5590]只(29431,320)好(29751,290)到(30041,390)屋(30431,350)顶(30781,340)找(31121,330)另(31451,360)一(31811,710)个(32521,260)梦(32781,1380)境(34161,860)
[40361,5059]女：睡(40361,300)梦(40661,310)中(40971,340)被(41311,340)敲(41651,320)醒 (41971,640)我(43041,320)还(43361,320)是(43681,350)不(44031,320)确(44351,330)定(44681,739)
[45790,5010]怎(45790,320)会(46110,340)有(46450,320)动(46770,350)人 (47120,330)旋(47450,320)律(47770,390)在(48160,370)对(48530,330)面(48860,340)的(49200,350)屋(49550,600)顶(50150,650)
[51200,5170]我(51200,320)悄(51520,330)悄(51850,350)关(52200,300)上(52500,340)门 (52840,810)带(53970,300)着(54270,320)希(54590,330)望(54920,320)上(55240,320)去(55560,810)
```


其中我们对某一句具体的歌词进行分析：

```
[24092,4969]男：半(24092,250)夜(24342,280)睡(24622,350)不(24972,300)着(25272,320)觉 (25592,650)把(26682,310)心(26992,330)情(27322,340)哼(27662,310)成(27972,320)歌(28292,769)
```

* [24092,4969]：24092毫秒是这句歌词的开始时间，4969是这句歌词的动画总时长
* 男：半(24092,250)，24092毫秒 是 歌词 “男：半(24092,250)”的开始时间，250毫秒是动画时长



我们做如下的定义：

* Lyrics 代表一首歌的歌词
* Sentence 代表具体的某一行
* Word 代表某一行中的某个字（可能不是单一字符，比如英语单词）

那么根据歌词的定义，即：

* 歌词由很多行组成
* 行由很多字组成
* 个字有开始时间与长度
* 行也有开始时间与长度

我们能写出如下的伪代码：

```swift
struct Lyrics {

    var artist: String? = nil

    var album: String? = nil

    var sentence: [Sentence] = []
}

extension Lyrics {

    struct Sentence {
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

解析歌词比较简单，只要用正则表达式进行提取，总共涉及到4个正则表达是

| 正则表达是 | 描述 |
| -- | -- |
| `\[(.*?):(.*?)\]` | 匹配标签，即得到 类似 `[ti:屋顶]`这样的结果 |
| `\[\d*,\d*\].+` | 匹配句子，由于`.`是匹配非换行符的任一字符，所以可以用来匹配出具体。得到的是类似 `[20,20]词：周杰伦(20,20)`的匹配结果 |
| `\[(\d*),(\d*)\]` | 匹配句子时间，对句子匹配的结果进行匹配，得到类似 `[20,20]`这样的结果 |
| `\((\d+),(\d+)\)` | 匹配单词时间，对句子匹配的结果进行匹配，得到一组时间，类似 `(24092,250)、(24342,280)、(24622,350) ...` |

那么解析歌词的思路

* 匹配所有的标签，根据标签的key不一样，得到具体某个标签的值，比如歌手、专辑
* 匹配所有的句子
* 匹配句子的开始时间与结束时间
* 匹配句子中的单词时间，注意计算index，因为单词在单词的时间之前

```swift
class QRCParser: NSObject {

    static let shared = QRCParser()

    private static let tagRegex = try? NSRegularExpression(pattern: #"\[(.*?):(.*?)\]"#, options: [])
    
    private static let lyricsRegex = try? NSRegularExpression(pattern: #"\[\d*,\d*\].+"#, options: [])
    
    private static let sentenseRegex = try? NSRegularExpression(pattern: #"\[(\d*),(\d*)\]"#, options: [])
    
    private static let wordRegex = try? NSRegularExpression(pattern: #"\((\d+),(\d+)\)"#, options: [])

    func parse(url: URL) -> Lyrics? {
        guard let data = try? Data(contentsOf: url),
            let content = String(data: data, encoding: .utf8) else {
            return nil
        }
        
        let contentRange = NSRange(location: 0, length: content.count)
        guard let matches = QRCParser.lyricsRegex?.matches(in: content, options: .withoutAnchoringBounds, range: contentRange),
            matches.count > 0 else {
            return nil
        }
        
        var lyrics = Lyrics()
        for match in matches {
            let line = content.subString(range: match.range)
            let lineRange = NSRange(location: 0, length: line.count)
            var lineStartTime: Int = 0
            var lineDuration: Int = 0
            var timeMatchResult: NSTextCheckingResult? = nil
            var lineOffsetIndex: String.Index = line.startIndex
            if let timeMatches = QRCParser.sentenseRegex?.matches(in: line, options: .withoutAnchoringBounds, range: lineRange),
                let timeMatch = timeMatches.first {
                let timestamp = line.subString(range: timeMatch.range).trimSquareBrackets()
                lineOffsetIndex = line.index(line.startIndex, offsetBy: timeMatch.range.length)
                let components = timestamp.components(separatedBy: ",")
                if components.count == 2 {
                    lineStartTime = Int(components[0]) ?? 0
                    lineDuration = Int(components[1]) ?? 0
                }
                timeMatchResult = timeMatch
            }
            
            var words: [Word] = []
            if let wordMatches = QRCParser.wordRegex?.matches(in: line, options: .withoutAnchoringBounds, range: lineRange) {
                for (index, wordMatch) in wordMatches.enumerated() {
                    
                    let wordTime = line.subString(range: wordMatch.range).trimCircleBrackets()
                    var wordContent: String = ""
                    if index == 0 {
                        if let timeMatch = timeMatchResult, timeMatch.range.location < line.count {
                            let end = line.index(line.startIndex, offsetBy: wordMatch.range.location)
                            wordContent = String(line[lineOffsetIndex ..< end])
                        }
                    } else {
                        let previousResult = wordMatches[index - 1]
                        let wordStartIndex = line.index(line.startIndex, offsetBy: previousResult.range.location + previousResult.range.length)
                        let WordEndIndex = line.index(line.startIndex, offsetBy: wordMatch.range.location)
                        wordContent = String(line[wordStartIndex ..< WordEndIndex])
                    }
                    
                    let wordTimeComponents = wordTime.split(separator: ",")
                    if wordTimeComponents.count == 2 {
                        let wordStart = Int(wordTimeComponents[0]) ?? 0
                        let wordDuration = Int(wordTimeComponents[1]) ?? 0
                        let word = Word(start: wordStart, duration: wordDuration, content: wordContent)
                        words.append(word)
                    }
                }
            }
            let sentence = Sentence(start: lineStartTime, duration: lineDuration, words: words)
            lyrics.sentences.append(sentence)
        }
        return lyrics
    }
}
```

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

