---
layout: post
title:  "iOS逆向手册"
date:   2020-05-14 10:30:10 +0800
tag: iOS
---

## 工具

| 工具 | 说明 |
| -- | -- |
| [iOS App Signer](https://dantheman827.github.io/ios-app-signer/) | 用于对ipa文件进行重签名 |
| [Hopper Disassembler](https://www.hopperapp.com/) | 查看与调试代码 |
| [MonekeyDev](https://github.com/AloneMonkey/MonkeyDev) | 神器，可以对砸过壳的应用做很多事情 |
| [iOS Images Extractor](https://github.com/devcxm/iOS-Images-Extractor) | 用于提取ipa中的图片资源 |
| [PP助手]() | 主要用来下载砸壳过的ipa，也用于打开SSH连接 |
| [爱思助手](https://www.i4.cn/) | 类似PP助手 |
| [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump) | 用于将ipa从越狱手机中导出，导出的ipa已砸壳 |

## 越狱

目前有多种越狱办法，笔者采用的是 [Unc0ver](https://unc0ver.dev/)，按照操作步骤就可以了，十分方便，并且支持最新的iOS 13.5

| 名称 | 支持iOS版本 | 说明 |
| -- | -- | -- |
| [Electra](https://coolstar.org/electra/) | iOS 11 - iOS 11.4.1 | 所有设备 |
| [Unc0ver](https://unc0ver.dev/) | iOS 11.0 - iOS 13.5 | - |
| [Chimera](https://chimera.sh/) | iOS 12.0 - iOS 1.2 以及 iOS 12.4 | 所有设备 |
| [Checkra1n](https://checkra.in/) | iOS 12.3 以上 | iPhone 5s 以上设备 |

越狱后手机上会多出来一个`Cydia`应用图标，接下来就装一些必备的插件了

* AFC2 - 激活PC端全路径访问。 AFC2需要添加 ABCydia/雷锋源 [https://apt.abcydia.com](https://apt.abcydia.com)
* FLEXing - 作者是Tanner Bennett，在Cydia自带的BigBoss源中就有，直接搜索就可以，安装后会重启SpringBoard，然后就可以长按导航栏唤出 FLEX的工具条了
* OpenSSH - 通过Terminal访问手机
* Frida - 添加 源 `build.frida.re`，下载合适的frida
* LookinLoader - 将Lookin集成到App中，连接到电脑后，可以用mac端的Lookin查看应用的UI结构，类似于Reveal

## 统计使用的第三方类库

使用class-dump导出头文件后，统计以 `PodsDummy_` 开头的

[WeChatShot](https://github.com/lefex/WeChatShot) 目录下 podlib/source，将main.py中的路径修改为头文件所在的路劲，执行 python main.py即可。

简单看了下源代码，其原理就是将CocoaPods所有第三方库拉取到本地存到数据库中，然后提取字符串中PodsDummy开头的字符串，在数据库中进行查找，再按照类库Star进行排序。


## 查看App应用内数据

有时候，我们需要查看应用内的数据。如果是自己开发的应用，直接使用Xcode就可以了。

对于越狱手机也十分简单，可以安装爱思助手之类的软件，直接可以在软件中流

#### FLEX

[FLEX](https://github.com/Flipboard/FLEX) 是Flipboard提供的一款强大的应用内调试工具。

![FLEX](/assets/images/2020/flexing.jpg)

* 查看视图层级，修改视图
* 查看在堆上的对象
* 查看应用内数据
* 监听应用网络请求，并提供curl复制
* 更多功能可以查看官网

#### Reveal


#### MonkeyDev

使用MonkeyDev可以安装第三方的Pod，比如使用[FLEX](https://github.com/Flipboard/FLEX)


由于 [GCDWebUploader](https://github.com/swisspol/GCDWebServer) 提供了比较友好的Web界面，在集成的过程中发现始终无法开启WebServer。

```objective-c
#import <GCDWebUploader.h>

static GCDWebUploader *_webUploader;

static __attribute__((constructor)) void entry(){
    NSLog(@"\n               🎉!!！congratulations!!！🎉\n👍----------------insert dylib success----------------👍");
    
    [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidFinishLaunchingNotification object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
        
        NSString* documentsPath = NSHomeDirectory();
        _webUploader = [[GCDWebUploader alloc] initWithUploadDirectory:documentsPath];
        NSError *error;
        [_webUploader startWithOptions:nil error:&error];
        NSLog(@"============== Visit %@ in your web browser", _webUploader.serverURL);        
#ifndef __OPTIMIZE__
        CYListenServer(6666);
#endif
    }];
}

```

于是一步一步Debug发现，_webUploader没有初始化成功，原因是 GCDWebUploader 使用了Bundle来加载网页资源，比较暴力的解法就是修改GCDWebUploader源代码，把 `[NSBundle bundleForClass:[GCDWebUploader class]]`替换为`[NSBundle mainBundle]`，然后把`GCDWebUploader.bundle`复制到应用内即可。

#### 越狱后重启失效

Unc0ver 不是完美越狱，所以会导致重启后越狱失效，解决方案就是重新越狱。

#### 下载的应用不弹出网络授权

有时候会遇到从App Store上下载的应用打开后，无法弹出网络授权的弹窗。解决方案就是，重启手机打开App后会弹出网络授权，再重新越狱。