---
layout: post
title:  "iOS常见的审核被拒理由"
date:   2019-01-16 21:33:10 +0800
tag: iOS杂货铺
---

文章收集笔者在开发iOS项目时遇到的AppStore提交审核被拒的理由以及解决方案，不定时更新


### Guideline 1.2 - Safety - User Generated Content


> Your app enables the display of user-generated content but does not have the proper precautions in place. 
>
> Next Steps 
>
> To resolve this issue, please revise your app to implement all of the following precautions:
>
> - Require that users agree to terms (EULA) and these terms must make it clear that there is no tolerance for objectionable content or abusive users 
> - A method for filtering objectionable content 
> - A mechanism for users to flag objectionable content 
> - A mechanism for users to block abusive users 
> - The developer must act on objectionable content reports within 24 hours by removing the content and ejecting the user who provided the offending content


原因：你的应用展示了用户生成的内容，但是没有合适的预防措施。

解决办法：实际上就是你的应用有用户生成的内容，但是没有举报相关的入口，以及可能缺少用户协议。

* 增加用户协议，并且在内容中明确提到敏感内容和用户的相关措施。 可以在用户登录时，需要用户勾选用户协议后，才允许登录
* 增加举报内容的入口
* 增加举报用户的入口
* 开发者必须在24小时内移除敏感内容。（这条苹果可能会认真审核....所以举报了就不要显示了...）

<hr> 


### Guideline 2.5.4 - Performance - Software Requirements

> Your app declares support for audio in the UIBackgroundModes key in your Info.plist, but we were unable to play any audible content when the app was running in the background.
>
> Next Steps
>
> The audio key is intended for use by apps that provide audible content to the user while in the background, such as music player or streaming audio apps. Please revise your app to provide audible content to the user while the app is in the background or remove the "audio" setting from the UIBackgroundModes key.

原因：应用在Info.plist中申明了 支持audio的UIBackgroundModes，但是在后台并不能播放声音

解决办法：在备注中说明如何测试，或者如果没有这个功能的话移除这个key的勾选。

<hr>


### Guideline 4.2.2 - Design - Minimum Functionality

> We noticed that your app only includes links, images, or content aggregated from the Internet with limited or no native iOS functionality. Although this content may be curated from the web specifically for your users, since it does not sufficiently differ from a mobile web browsing experience, it is not appropriate for the App Store. 
>
> Next Steps 
>
> We encourage you to review your app concept and work towards creating an app that offers customers an engaging and lasting experience that also meets the App Store’s high expectations for quality and functionality. 
> 
> Apple Developer includes a variety of design and development resources. Download iOS templates from Apple UI Design Resources, learn more about crafting intuitive, well-designed apps with the Design Video collection, and review the iOS Human Interface Guidelines for best practices to follow when designing apps for the App Store.


原因：苹果认为你的应用只是简单的从网络上收集链接、图片等来展示，这些功能完全可以通过浏览器来完成。苹果认为这样的应用达不到上线AppStore的标准。

解决办法：增加一些用户产生内容的功能

<hr> 

### Guideline 4.2.3 - Design - Minimum Functionality

> We were required to install QQ and WeChat￼ before we could use your app. Apps should be able to run on launch, without requiring additional apps to be installed. 
>
> Next Steps 
>
> To resolve this issue, please revise your app to ensure that users can use it upon launch. If your app requires authentication before use, please use methods that can authenticate users from within your app. 
>
> Please see attached screenshot for details.

原因：我们需要安装QQ和微信才能使用你的应用。应用应该不需要依赖其他的应用才能正常使用。

解决办法：这是比较长遇到的一个被拒原因，也是非常好解决。就是苹果认为你的应用不能依赖于其他应用才能运行，解决办法就是如果没有装QQ或者微信就不要显示QQ或者微信按钮。

如果你的应用只能通过微信登录，那么在审核的时候可以加一个手机号登录，服务端写死一个手机号与验证码，供审核人员使用。当然建议是直接支持手机号登录。


<hr> 

### Guidline 4.1 - Copycats

> Your app or its metadata appears to contain misleading content.
>
> Specifically, your app includes content that resembles xxxxx

原因：苹果认为你的应用抄袭了AppStore上的其他应用

解决办法：重新设计你的应用UI


<hr>