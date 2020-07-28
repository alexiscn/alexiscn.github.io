---
layout: post
title:  "给WXActionSheet增加暗黑模式"
date:   2020-07-28 15:05:10 +0800
tag: iOS
---

之前写了一个[仿微信ActionSheet](/2019/09/07/ios_wechat_navigationbar.html)，在自己的项目中也经常用到。iOS13中推出了Dark Mode后，一直没有时间做适配[WXActionSheet](https://github.com/alexiscn/WXActionSheet)。最近稍微空闲了就给它增加了暗黑主题支持，主要增加了一个样式枚举，一个样式配置类。效果图如下：

![](/assets/images/2020/wxactionsheet_dark_mode.png)

```swift
public enum WXActionSheetStyle {
    
    /// The system style. avaiable at iOS 13.0
    @available(iOS 13, *)
    case system
    /// The light style.
    case light
    /// The dark style.
    case dark
    /// The custom style.
    case custom(Appearance)
}
```

提供了多种样式：

- `system` 跟随系统样式，仅在iOS13之后的系统中生效。
- `light` 亮色主题，在iOS12之前是默认主题
- `dark` 黑色主题
- `custom(Appearance)` 自定义样式

自定义的样式即提供了给用户设置样式的入口，可配置的属性值如下：

```swift
struct Appearance {
        
    /// The background color of dimming background view.
    public var dimmingBackgroundColor: UIColor = UIColor(white: 0, alpha: 0.5)
    
    /// The background color for the container view.
    public var containerBackgroundColor: UIColor = .white
    
    /// The text color for the title.
    public var titleColor: UIColor = UIColor(white: 0, alpha: 0.5)
    
    /// The height of button. 56 by default.
    public var buttonHeight: CGFloat = 56.0
    
    /// The background color of button in normal state.
    public var buttonNormalBackgroundColor = UIColor(white: 1, alpha: 1)

    /// The background color of button in highlighted state.
    public var buttonHighlightBackgroundColor = UIColor(white: 248.0/255, alpha: 1)
    
    /// Title color for normal buttons, default to UIColor.black
    public var buttonTitleColor = UIColor.black
    
    /// Title color for destructive button, default to UIColor.red
    public var destructiveButtonTitleColor = UIColor.red
    
    /// The separator line color.
    public var separatorLineColor: UIColor = UIColor(white: 0, alpha: 0.05)
    
    /// The  background color for separator between CancelButton and other buttons.
    public var separatorColor: UIColor = UIColor(white: 242.0/255, alpha: 1.0)
    
    /// A Boolean value indicating whether enable blur effect.
    public var enableBlurEffect: Bool = false
    
    public var effect: UIVisualEffect = UIBlurEffect(style: .light)
    
    public init() { }
}
```

同时在`WXActionSheetStyle`中暴露了默认的light和dark样式，可以在外部修改其属性值来全局修改默认样式。

```swift
public enum WXActionSheetStyle {
    @available(iOS 13, *)
    case system
    case light
    case dark
    case custom(Appearance)

    /// The default dark appearance for light style. You can change any property to create new light style.
    public static var lightAppearance: Appearance = {
        var appearance = Appearance()
        appearance.dimmingBackgroundColor = UIColor(white: 0, alpha: 0.5)
        appearance.containerBackgroundColor = .white
        appearance.titleColor = UIColor(white: 0, alpha: 0.5)
        appearance.buttonHeight = 56.0
        appearance.buttonNormalBackgroundColor = UIColor(white: 1, alpha: 1)
        appearance.buttonHighlightBackgroundColor = UIColor(white: 248.0/255, alpha: 1)
        appearance.buttonTitleColor = .black
        appearance.destructiveButtonTitleColor = .red
        appearance.separatorLineColor = UIColor(white: 0, alpha: 0.1)
        appearance.separatorColor = UIColor(white: 247.0/255, alpha: 1.0)
        appearance.enableBlurEffect = false
        return appearance
    }()

    // ...
}
```

暗黑模式在 0.6.0 版本以上可用。在后续的版本中还会增加可滚动的支持，类似微信中打开网页后，点击...出来的ActionSheet，一般用于分享场景，在一般项目中也较为常用。