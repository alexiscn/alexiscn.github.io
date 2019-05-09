---
layout: post
title:  "iOS自定义键盘播放声音"
date:   2018-02-07 12:32:10 +0800
tag: iOS杂货铺
---

自定义的键盘可以播放自定义的音频文件。有几个注意点记录下：

1、需要设置Info.plist中的RequestsOpenAccess为YES，如下图所示

![image](/assets/images/custom-keyboard-info-plist.jpg)

2、需要用户添加自定义键盘，并且在键盘设置中，打开 `Allow Full Acess`的开关

3、播放声音的代码片段如下： 


```swift
let button = UIButton(type: .system)
button.setTitle("play sound", for: .normal)
button.translatesAutoresizingMaskIntoConstraints = false
self.view.addSubview(button)

button.centerXAnchor.constraint(equalTo: self.view.centerXAnchor).isActive = true
button.centerYAnchor.constraint(equalTo: self.view.centerYAnchor).isActive = true
button.addTarget(self, action: #selector(handleButtonTapped(_:)), for: .touchUpInside)
```


```swift
@objc func handleButtonTapped(_ sender: UIButton) {
    DispatchQueue.main.async {
        guard let path = Bundle.main.path(forResource: "sound3", ofType: "mp3") else {
            return
        }
        let url = NSURL(fileURLWithPath: path)
        var theSoundID: SystemSoundID = 0
        AudioServicesCreateSystemSoundID(url, &theSoundID)
        AudioServicesPlaySystemSound(theSoundID)
    }
}
```

注意音频文件不能超过30秒，需要时mp3或者支持的文件格式