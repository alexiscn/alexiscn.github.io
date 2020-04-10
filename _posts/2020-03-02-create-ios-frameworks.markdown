---
layout: post
title:  "同时支持CocoaPods、Carthage、SPM的开源类库"
date:   2020-03-02 19:32:10 +0800
tag: 经验分享
---

有时候我们封装了一些不错的可复用的类库，希望发布到Github当轮子用，希望类库同时支持 `CocoaPods`、`Carthage` 和 `Swift Package Manager`方式的集成到iOS项目中。

### 第一步：创建Github项目

假设我们要封装的类库的名字叫 `AwesomeKit`，那么首先在Github上创建对应的项目，并将其Clone到本地。

### 第二步：创建Framework项目

创建iOS 动态类库，如下图所示：

![](/assets/images/2020/awesomekit_create_framework@2x.png)

由于`Carthage`需要项目的`Scheme`是共享的，所以我们一开始创建的是类库项目，这样默认`Scheme`是共享的。

![](/assets/images/2020/awesomekit_shared_scheme@2x.png)

然后重命名文件夹的名字为 `Sources`，因为Swift Package Manager的默认存放源代码的名字就是`Sources`，当然你也可以不用修改，在`Package.swift`中指定`Path`即可。

![](/assets/images/2020/awesomekit_rename@2x.png)


### 第二步：添加podspec

在`AwesomeKit`跟目录下新建 `AwesomeKit.podspec`，用于支持CocoaPods发布。

```bash
$ vim AwesomeKit.podspec
```

文件内容如下，可以对应做一些修改，更多的语法可以参考CocoaPods的[官方文档](https://guides.cocoapods.org/syntax/podspec.html)

```
Pod::Spec.new do |s|
  s.name         = 'AwesomeKit'
  s.version      = '0.0.1'
  s.license = 'MIT'
  s.requires_arc = true
  s.source = { :git => 'https://github.com/alexiscn/AwesomeKit.git', :tag => s.version.to_s }

  s.summary         = 'AwesomeKit'
  s.homepage        = 'https://github.com/alexiscn/AwesomeKit'
  s.license         = { :type => 'MIT' }
  s.author          = { 'alexiscn' => 'alexiscn@example.com' }
  s.platform        = :ios
  s.swift_version   = '5.0'
  s.source_files    =  'Sources/**/*.{swift}'
  s.ios.deployment_target = '12.0'
  
end
```

注意这里指定的路径是 `Sources`。

### 第四步：添加Package.swift

在`AwesomeKit`跟目录下运行如下命令用来创建Swift Package Manager所需要的文件

```bash
$ swift package init
```

会看到如下的提示信息：

```bash
Creating library package: AwesomeKit
Creating Package.swift
Creating README.md
Creating .gitignore
Creating Tests/
Creating Tests/LinuxMain.swift
Creating Tests/AwesomeKitTests/
Creating Tests/AwesomeKitTests/AwesomeKitTests.swift
Creating Tests/AwesomeKitTests/XCTestManifests.swift
```

由于我们创建的是iOS的类库，所以需要编辑Package.swift，指定其运行的平台是 iOS，最低支持的版本等。

```swift
// swift-tools-version:5.1
// The swift-tools-version declares the minimum version of Swift required to build this package.
import PackageDescription

let package = Package(
    name: "AwesomeKit",
    platforms: [.iOS(.v12)],
    products: [
        // Products define the executables and libraries produced by a package, and make them visible to other packages.
        .library(
            name: "AwesomeKit",
            targets: ["AwesomeKit"]),
    ],
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        // .package(url: /* package url */, from: "1.0.0"),
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages which this package depends on.
        .target(
            name: "AwesomeKit",
            dependencies: [],
            path: "Sources"),
        .testTarget(
            name: "AwesomeKitTests",
            dependencies: ["AwesomeKit"]),
    ]
)
```

### 第五步：创建Example

在完成类库的封装后，就可以写一个例子引用封装的轮子了，推荐使用Pod的方式。创建`AwesomeKitExample` iOS项目，将其放到根目录下的`Example`文件夹中。像平时使用Pod的方式，添加Podfile，执行 `pod install`。

此时目录结构大致如下：

```
.
├── AwesomeKit.podspec
├── AwesomeKit.xcodeproj
├── Example
│   ├── AwesomeKitExample
│   │   ├── AppDelegate.swift
│   │   ├── Assets.xcassets
│   │   │   ├── AppIcon.appiconset
│   │   │   │   └── Contents.json
│   │   │   └── Contents.json
│   │   ├── Base.lproj
│   │   │   ├── LaunchScreen.storyboard
│   │   │   └── Main.storyboard
│   │   ├── Info.plist
│   │   ├── SceneDelegate.swift
│   │   └── ViewController.swift
│   ├── AwesomeKitExample.xcodeproj
│   ├── AwesomeKitExample.xcworkspace
│   ├── Podfile
│   ├── Podfile.lock
│   └── Pods
│       ├── Headers
│       ├── Local\ Podspecs
│       │   └── AwesomeKit.podspec.json
│       ├── Manifest.lock
│       ├── Pods.xcodeproj
│       └── Target\ Support\ Files
│           ├── AwesomeKit
│           │   ├── AwesomeKit.debug.xcconfig
│           │   └── AwesomeKit.release.xcconfig
│           └── Pods-AwesomeKitExample
│               ├── Pods-AwesomeKitExample-Info.plist
│               ├── Pods-AwesomeKitExample-acknowledgements.markdown
│               ├── Pods-AwesomeKitExample-acknowledgements.plist
│               ├── Pods-AwesomeKitExample-dummy.m
│               ├── Pods-AwesomeKitExample-umbrella.h
│               ├── Pods-AwesomeKitExample.debug.xcconfig
│               ├── Pods-AwesomeKitExample.modulemap
│               └── Pods-AwesomeKitExample.release.xcconfig
├── Package.swift
├── README
├── README.md
├── Sources
│   ├── AwesomeKit.h
│   ├── AwesomeKit.swift
│   └── Info.plist
└── Tests
    ├── AwesomeKitTests
    │   ├── AwesomeKitTests.swift
    │   └── XCTestManifests.swift
    └── LinuxMain.swift
```


### 第六步：提交到Github


```bash
$ git add .
$ git commit -m 'initial commit'
$ git push
$ git tag 0.0.1
$ git push --tags
```

##### CocoaPods

将类库发布到CocoaPods Trunk，首次提交时可能需要让你注册，按照提示完成即可

```bash
$ pod trunk push AwesomeKit.podspec
```

##### Carthage

只需要验证一下 Carthage即可

```bash
$ carthage build --no-skip-current
```

查看Carthage/Build目录下是否有 AwesomeKit.framework

### 一些注意点

由于 `Swift Package Manager` 目前不支持Resource，所以当类库中有图片资源时，可以采取妥协的办法，即把图片先转换成Base64字符串，硬编码到源代码中，然后读取成Data，最后转成UIImage。