---
layout: post
title:  "iOS逆向: 一步一步实现微信导航栏"
date:   2019-09-07 11:33:10 +0800
tag: WeChat
---


## 准备工具

- 某某助手上下载 微信.ipa
- class-dump
- hopper disassembler

## 导出头文件

```
$ class-dump WeChat.app -H -o headers
```

找到我们感兴趣的几个头文件

- MMUINavigationBar.h
- FakeNavigationBar.h
- MMUINavigationController.h
- MMUIViewController.h

## MMUINavigationBar

查看 MMUINavigationBar 头文件，发现MMUINavigationBar 是继承自 UINavigationBar的自定义导航栏，并且暴露出来的方法也比较少，猜测微信仅仅是修改了系统导航栏的某些属性。

```objective-c
@interface MMUINavigationBar : UINavigationBar
{
    UIView *_effectSubview;
}

@property(retain, nonatomic) UIView *effectSubview; // @synthesize effectSubview=_effectSubview;
- (void).cxx_destruct;
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

- (void).cxx_destruct;
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

// Remaining properties
@property(readonly, copy) NSString *debugDescription;
@property(readonly, copy) NSString *description;
@property(readonly) unsigned long long hash;
@property(readonly) Class superclass;

@end
```

## MMUIViewController

MMUIViewController 是微信中所有ViewController的基类。由于头文件代码较多，我仅将与导航栏相关的代码保留，其余都删掉了。

```objective-c
@interface MMUIViewController : UIViewController <IUiUtilExt, MMUIViewControllerDelegate, UIGestureRecognizerDelegate>
{
    _Bool m_isPopByClickingURL;
    MMLoadingView *m_loadingViewX;
    unsigned int m_uiVcType;
    UILabel *m_newsTitleRecordLabel;
    NSMutableArray *m_fullScreenViews;
    _Bool m_bAnimated;
    _Bool m_bIsBeingPoped;
    _Bool m_bInteractivePopEnabled;
    _Bool m_bDisableAdjustInsetAndOffset;
    double lastScreenWidth;
    UINavigationController *m_navigationController;
    MMTitleView *m_baseTitleView;
    NSMutableDictionary *m_dicDeepLink;
    NSMutableDictionary *m_dicContentInsetAutolayout;
    NSMutableArray *m_arrEndUserOpInfo;
    MMDelegateProxy<UIGestureRecognizerDelegate> *m_interactivePopGestureRecognizerDelegate;
    UIBarButtonItem *m_leftBarBtnItem;
    UIBarButtonItem *m_rightBarBtnItem;
    UIColor *m_titleColor;
    MMUINavigationBar *fakeNaviView;
    UIView *m_navHeaderView;
    UIView *m_navSepLine;
    double m_navWidth;
    NSString *m_navTitle;
    NSString *m_navSubTitle;
    _Bool m_navLoading;
    UIView *m_navRightView;
    double m_navTitleOffset;
    _Bool m_navCustomView;
    _Bool _isDuringInteractivePop;
    _Bool _m_bStopPopWhenDeleteContact;
    UIView *bottomView;
    UIViewController *_presentingModalViewController;
    UIViewController *_presentedModalViewController;
    MMNavBarInteractiveConfig *_navBarInteractiveConfig;
    WCEventTrackingSystemConfig *_trackingSystemConfig;
    UIPanGestureRecognizer *_scrollViewInteractivePanGesture;
}

@property(retain, nonatomic) UIPanGestureRecognizer *scrollViewInteractivePanGesture; // @synthesize scrollViewInteractivePanGesture=_scrollViewInteractivePanGesture;
@property(retain, nonatomic) WCEventTrackingSystemConfig *trackingSystemConfig; // @synthesize trackingSystemConfig=_trackingSystemConfig;
@property(retain, nonatomic) MMNavBarInteractiveConfig *navBarInteractiveConfig; // @synthesize navBarInteractiveConfig=_navBarInteractiveConfig;
@property(nonatomic) _Bool m_bStopPopWhenDeleteContact; // @synthesize m_bStopPopWhenDeleteContact=_m_bStopPopWhenDeleteContact;
@property(nonatomic) _Bool isDuringInteractivePop; // @synthesize isDuringInteractivePop=_isDuringInteractivePop;
@property(nonatomic) _Bool m_bAnimating; // @synthesize m_bAnimating=_m_bAnimating;
@property(nonatomic) __weak UIViewController *presentedModalViewController; // @synthesize presentedModalViewController=_presentedModalViewController;
@property(nonatomic) __weak UIViewController *presentingModalViewController; // @synthesize presentingModalViewController=_presentingModalViewController;
@property(nonatomic) _Bool m_bIsBeingInteractivePop; // @synthesize m_bIsBeingInteractivePop;
@property(retain, nonatomic) NSMutableArray *m_arrEndUserOpInfo; // @synthesize m_arrEndUserOpInfo;
@property(nonatomic) _Bool m_bDisableAdjustInsetAndOffset; // @synthesize m_bDisableAdjustInsetAndOffset;
@property(nonatomic) _Bool m_bInteractivePopEnabled; // @synthesize m_bInteractivePopEnabled;
@property(nonatomic) _Bool m_bIsBeingPoped; // @synthesize m_bIsBeingPoped;
@property(nonatomic) _Bool m_bAnimated; // @synthesize m_bAnimated;
@property(retain, nonatomic) UIView *bottomView; // @synthesize bottomView;
@property(retain, nonatomic) UILabel *m_newsTitleRecordLabel; // @synthesize m_newsTitleRecordLabel;
@property(nonatomic) unsigned int m_uiVcType; // @synthesize m_uiVcType;
@property(retain, nonatomic) MMLoadingView *loadingViewX; // @synthesize loadingViewX=m_loadingViewX;
- (void).cxx_destruct;
- (id)mmNavigationController:(id)arg1 interactionControllerForAnimationController:(id)arg2;
- (id)mmNavigationController:(id)arg1 animationControllerForOperation:(long long)arg2 fromViewController:(id)arg3 toViewController:(id)arg4;
- (_Bool)gestureRecognizer:(id)arg1 shouldBeRequiredToFailByGestureRecognizer:(id)arg2;
- (_Bool)gestureRecognizer:(id)arg1 shouldRequireFailureOfGestureRecognizer:(id)arg2;
- (_Bool)shouldInteractivePop;
- (_Bool)gestureRecognizer:(id)arg1 shouldRecognizeSimultaneouslyWithGestureRecognizer:(id)arg2;
- (_Bool)gestureRecognizerShouldBegin:(id)arg1;
- (_Bool)interactivePopGestureRecognizerShouldBegin:(id)arg1;
- (_Bool)gestureRecognizer:(id)arg1 shouldReceiveTouch:(id)arg2;
- (void)resignSubviewResponder:(id)arg1;
- (void)viewWillInteractivePop;
- (void)viewDidBeInteractivePoped;
- (void)viewWillBeInteractivePoped;
- (void)viewWillDismiss:(_Bool)arg1;
- (void)viewDidPresent:(_Bool)arg1;
- (void)viewWillPresent:(_Bool)arg1;
- (void)viewDidPop:(_Bool)arg1;
- (void)viewWillPop:(_Bool)arg1;
- (void)viewDidPush:(_Bool)arg1;
- (void)viewWillPush:(_Bool)arg1;
- (void)viewDidBeDismissed:(_Bool)arg1;
- (void)viewWillBeDismissed:(_Bool)arg1;
- (void)viewDidBePresented:(_Bool)arg1;
- (void)viewWillBePresented:(_Bool)arg1;
- (void)viewDidBePoped:(_Bool)arg1;
- (void)viewWillBePoped:(_Bool)arg1;
- (void)viewDidBePushed:(_Bool)arg1;
- (void)viewWillBePushed:(_Bool)arg1;
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
- (void)traitCollectionDidChange:(id)arg1;
- (_Bool)showNavigationBarSepLine;
- (_Bool)navigationBarBlurEffect;
- (id)navigationBarBackgroundColor;
- (_Bool)useTransparentNavibar;
- (void)updateStatusBarColor;
- (_Bool)useBlackStatusbar;
- (_Bool)hidesStatusBar;
- (void)didDisappearToSearchController;
- (void)willDisappearToSearchController;
- (void)didAppearFromSearchController;
- (void)willAppearFromSearchController;
- (void)viewDidDisappear:(_Bool)arg1;
- (void)viewWillDisappear:(_Bool)arg1;
- (void)viewDidPopOrDismiss:(_Bool)arg1;
- (void)viewWillPopOrDismiss:(_Bool)arg1;
- (void)viewDidBePushOrPresent:(_Bool)arg1;
- (void)viewWillBePushOrPresent:(_Bool)arg1;
- (void)willAnimateRotationToInterfaceOrientation:(long long)arg1 duration:(double)arg2;
- (void)protectStatusBarFromBeingFuckedByForeGround:(SEL)arg1;
- (void)setStatusBarFontBlack;
- (void)setStatusBarFontWhite;
- (void)setStatusBarHidden:(_Bool)arg1 withAnimation:(long long)arg2;
- (void)setStatusBarHidden:(_Bool)arg1;
- (void)setTopBarsHidden:(_Bool)arg1 animated:(_Bool)arg2;
- (void)changeTopBarsHiddenAnimated:(_Bool)arg1;
- (double)tableView:(id)arg1 heightForFooterInSection:(long long)arg2;
- (double)tableView:(id)arg1 heightForHeaderInSection:(long long)arg2;
- (id)tableView:(id)arg1 viewForFooterInSection:(long long)arg2;
- (id)tableView:(id)arg1 viewForHeaderInSection:(long long)arg2;
- (void)setTitleOnly:(id)arg1;
- (void)willDismissAndShow;
- (void)setTitleInterfaceOritation:(long long)arg1;
- (void)reloadTitleView;
- (double)getRightBarButtonWidth;
- (double)getLeftBarButtonWidth;
- (double)adjustedStatusBarHeight;
- (_Bool)hasTitle;
- (void)setTitleView:(id)arg1;
- (void)setTitle:(id)arg1 subTitle:(id)arg2 leftLoading:(_Bool)arg3 rightView:(id)arg4 titleOffset:(double)arg5;
- (void)setTitle:(id)arg1 subTitle:(id)arg2 leftLoading:(_Bool)arg3 rightView:(id)arg4;
- (void)setTitle:(id)arg1 leftLoading:(_Bool)arg2;
- (void)setTitle:(id)arg1 subTitle:(id)arg2;
- (void)setTitle:(id)arg1 rightView:(id)arg2;
- (id)getTitleColor;
- (void)setTitleColor:(id)arg1;
- (void)setTitle:(id)arg1;
- (void)willShow;
- (void)willDisshow;
- (void)didDisshow;
- (void)didAppear;
- (void)willDisappear;
- (void)adjustView;
- (void)willAppear;
- (void)setIsPopByClickingURL;
- (void)restoreNavigationBarBkg;
- (void)removeNavigationBarBkg;
- (void)onMainWindowFrameChanged;
- (void)viewDidLayoutSubviews;
- (void)viewDidTransitionToNewSize;
- (void)setAutolayoutTopOffset:(double)arg1 forView:(id)arg2;
- (double)getContentViewYforTranslucentNaviBar;
- (void)observeValueForKeyPath:(id)arg1 ofObject:(id)arg2 change:(id)arg3 context:(void *)arg4;
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
- (void)internalHandleFade:(id)arg1;
- (void)onScrollViewInteractivePan:(id)arg1;
- (void)onScrollViewContentOffsetChanged:(struct CGPoint)arg1;
- (void)viewDidLayoutSubviewsInNavBar;
- (void)viewWillDisappearInNavBar:(_Bool)arg1;
- (void)viewWillAppearInNavBar:(_Bool)arg1;
@property(readonly, nonatomic) UIView *transitionRootView; // @dynamic transitionRootView;

- (void)setWCBizAuthTitle:(id)arg1;

@end

```

比较感兴趣的是如下的方法

- (_Bool)useTransparentNavibar;
- (_Bool)navigationBarBlurEffect;