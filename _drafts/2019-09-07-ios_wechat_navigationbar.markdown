---
layout: post
title:  "iOS逆向: 一步一步实现微信导航栏"
date:   2019-09-07 11:33:10 +0800
tag: WeChat
---

iOS项目开发中，难免被iOS导航栏折，比较折腾的地方在:

- 自定义返回图标
- 纯色导航栏与系统导航栏的平滑切换
- 系统导航栏与隐藏导航栏的平滑切换
- 一些特立独行的PM或UI想要扩大导航栏的高度，比如50pt（系统默认是 44pt）

> 而且产品经常会飘来一句话：“咦，微信的导航栏怎么这么平滑”

## 常见自定义导航栏 

### 设置背景透明

```swift
navigationController?.navigationBar.setBackgroundImage(UIImage(), for: .default)
```

### 隐藏导航栏底部黑线

```swift
navigationController?.navigationBar.shadowImage = UIImage()
```

### 自定义返回按钮图标

```swift
navigationController?.navigationBar.backIndicatorImage = UIImage(named: "icons_outlined_back")
navigationController?.navigationBar.backIndicatorTransitionMaskImage = UIImage(named: "icons_outlined_back")
```

## 准备工具

- 某某助手上下载 微信.ipa
- class-dump
- hopper disassembler
- [可选]一台越狱设备

## 导出头文件

```sh
$ class-dump WeChat.app -H -o headers
```

找到我们感兴趣的几个头文件

- MMUINavigationBar.h
- FakeNavigationBar.h
- MMUINavigationController.h
- MMUIViewController.h

此时大概的猜测是：微信使用的是系统导航栏，然后在页面中进行了特殊处理以上遇到的问题

## MMUINavigationBar

查看 MMUINavigationBar 头文件，发现MMUINavigationBar 是继承自 UINavigationBar的自定义导航栏，并且暴露出来的方法也比较少，猜测微信仅仅是修改了系统导航栏的某些属性。

```objective-c
@interface MMUINavigationBar : UINavigationBar
{
    UIView *_effectSubview;
}

@property(retain, nonatomic) UIView *effectSubview; // @synthesize effectSubview=_effectSubview;
- (id)hitTest:(struct CGPoint)arg1 withEvent:(id)arg2;
- (void)setAlpha:(double)arg1;
- (void)setBarStyle:(long long)arg1;
- (void)setTranslucent:(_Bool)arg1;
- (id)findBlurEffectView:(id)arg1;
- (id)findHiddenView:(id)arg1;
- (void)adjustShadowView;
- (void)layoutSubviews;
@end
```

## FakeNavigationBar

FakeNavigationBar 是继承自 MMUINavigationBar，提供了一个可以添加subView的方法。从名字上看是“假”的导航栏，那么微信有可能是用假的导航栏盖在真的导航栏上面，能想到的场景是朋友圈滑动时，导航栏从全透明切换到系统导航栏。

```objective-c
@interface FakeNavigationBar : MMUINavigationBar
{
}
- (void)layoutSubviews;
- (void)didAddSubview:(id)arg1;
@end
```

## MMUINavigationController

这个比较有意思了。

```objective-c
@interface MMUINavigationController : UINavigationController <UINavigationControllerDelegate>
{
    UIViewController *_popingViewController;
}

- (id)navigationController:(id)arg1 interactionControllerForAnimationController:(id)arg2;
- (id)navigationController:(id)arg1 animationControllerForOperation:(long long)arg2 fromViewController:(id)arg3 toViewController:(id)arg4;
- (void)layoutViewsForTaskBar;
- (void)viewWillLayoutSubviews;
- (void)viewDidLoad;
- (void)viewWillAppear:(_Bool)arg1;
- (void)setNavigationBarHidden:(_Bool)arg1 animated:(_Bool)arg2;
- (id)popViewControllerAnimated:(_Bool)arg1;
- (id)DispatchPopViewControllerAnimated:(_Bool)arg1;
- (id)initWithRootViewController:(id)arg1;
- (id)init;

@end
```

## MMUIViewController

`MMUIViewController` 是微信中所有ViewController的基类。由于头文件代码较多，我仅将与导航栏相关的代码保留，其余都删掉了。

```objective-c
@interface MMUIViewController : UIViewController <IUiUtilExt, MMUIViewControllerDelegate, UIGestureRecognizerDelegate>
{
    unsigned int m_uiVcType;
    UINavigationController *m_navigationController;
    MMTitleView *m_baseTitleView;
    MMDelegateProxy<UIGestureRecognizerDelegate> *m_interactivePopGestureRecognizerDelegate;
    UIBarButtonItem *m_leftBarBtnItem;
    UIBarButtonItem *m_rightBarBtnItem;
    UIColor *m_titleColor;
    MMUINavigationBar *fakeNaviView;
    UIView *m_navHeaderView;
    UIView *m_navSepLine;
    NSString *m_navTitle;
    NSString *m_navSubTitle;
    _Bool m_navLoading;
    UIView *m_navRightView;
    double m_navTitleOffset;
    _Bool m_navCustomView;
    MMNavBarInteractiveConfig *_navBarInteractiveConfig;
}

@property(retain, nonatomic) UIPanGestureRecognizer *scrollViewInteractivePanGesture; // @synthesize scrollViewInteractivePanGesture=_scrollViewInteractivePanGesture;
@property(retain, nonatomic) WCEventTrackingSystemConfig *trackingSystemConfig; // @synthesize trackingSystemConfig=_trackingSystemConfig;
@property(retain, nonatomic) MMNavBarInteractiveConfig *navBarInteractiveConfig; // @synthesize navBarInteractiveConfig=_navBarInteractiveConfig;
@property(retain, nonatomic) NSMutableArray *m_arrEndUserOpInfo; // @synthesize m_arrEndUserOpInfo;
@property(nonatomic) unsigned int m_uiVcType; // @synthesize m_uiVcType;

- (id)mmNavigationController:(id)arg1 interactionControllerForAnimationController:(id)arg2;
- (id)mmNavigationController:(id)arg1 animationControllerForOperation:(long long)arg2 fromViewController:(id)arg3 toViewController:(id)arg4;
- (_Bool)gestureRecognizerShouldBegin:(id)arg1;
- (_Bool)interactivePopGestureRecognizerShouldBegin:(id)arg1;
- (void)onNavigationBarHiddenChanged;
- (void)onNavigationBarAlphaChanged;
- (void)removeFakeNaviView;
- (void)restoreNavigationBar;
- (void)internalAddFakeNaviView:(id)arg1;
- (void)addFakeNaviView;
- (_Bool)useCustomNavibar;
- (void)restoreNavHeaderIfNeed;
- (id)getNavBarSepLine;
- (void)initNavibarSepLine;
- (void)initNavHeaderIfNeed;
- (id)getBarButtonWithImageName:(id)arg1 target:(id)arg2 action:(SEL)arg3 style:(unsigned long long)arg4 accessibility:(id)arg5 color:(id)arg6;
- (id)getBarButtonWithTitle:(id)arg1 target:(id)arg2 action:(SEL)arg3 style:(unsigned long long)arg4 color:(id)arg5;
- (id)getBarButtonWithImageName:(id)arg1 target:(id)arg2 action:(SEL)arg3 style:(unsigned long long)arg4 accessibility:(id)arg5;
- (id)getBarButtonWithTitle:(id)arg1 target:(id)arg2 action:(SEL)arg3 style:(unsigned long long)arg4;
- (_Bool)useWhiteForegroundColor;
- (id)navigationTitleColor;
- (void)onNavigationBarBackgroundColorChange;
- (_Bool)showNavigationBarSepLine;
- (_Bool)navigationBarBlurEffect;
- (id)navigationBarBackgroundColor;
- (_Bool)useTransparentNavibar;
- (void)updateStatusBarColor;
- (_Bool)useBlackStatusbar;
- (_Bool)hidesStatusBar;
- (void)protectStatusBarFromBeingFuckedByForeGround:(SEL)arg1;
- (void)setStatusBarFontBlack;
- (void)setStatusBarFontWhite;
- (void)setStatusBarHidden:(_Bool)arg1 withAnimation:(long long)arg2;
- (void)setStatusBarHidden:(_Bool)arg1;
- (void)setTopBarsHidden:(_Bool)arg1 animated:(_Bool)arg2;
- (void)changeTopBarsHiddenAnimated:(_Bool)arg1;
- (void)setTitleOnly:(id)arg1;
- (void)setTitleInterfaceOritation:(long long)arg1;
- (void)reloadTitleView;
- (double)adjustedStatusBarHeight;
- (_Bool)hasTitle;
- (void)setTitleView:(id)arg1;
- (void)setTitleColor:(id)arg1;
- (void)setTitle:(id)arg1;
- (void)restoreNavigationBarBkg;
- (void)removeNavigationBarBkg;
- (double)getContentViewYforTranslucentNaviBar;
- (void)updateNavibarSepline;
- (void)adjustViewAndNavBarRect;
- (id)titleView;
- (double)navigationBarHeight;
- (double)statusBarHeight;
- (void)setNavigationBarAlpha:(double)arg1 withTitleIncluded:(_Bool)arg2;
- (void)setNavigationBarY:(double)arg1;
- (void)restoreNavigationBarToFullSize;
- (void)restoreNavigationBarToFullSizeAnimatedWithDuration:(double)arg1;
- (void)restoreNavigationBarToFullSizeOnScrollToTop;
- (void)updateFadeBkgAlpha;
- (void)viewDidLayoutSubviewsInNavBar;
- (void)viewWillDisappearInNavBar:(_Bool)arg1;
- (void)viewWillAppearInNavBar:(_Bool)arg1;

@end

```

比较感兴趣的是如下的方法

- (_Bool)useTransparentNavibar;
- (_Bool)navigationBarBlurEffect;

## 猜测原理

微信导航栏的猜测

- 使用了系统的导航栏
- 使用了假的UIView作为有特殊


## 实现 MMNavigationBar

## 实现 MMNavigationController

## 实现 MMViewController
