---
layout: post
title:  "openURLContexts文件权限问题"
date: 2021-04-29 18:02:10 +0800
tag: ios
---

最近在给`Filebox`增加从AirDrop接收文件的时候遇到一个问题，接收到的音频文件在后台播放的时候提示`NSFileReadNoPermissionError`，百思不得其解，因为在Debug时完全可以正常播放，锁屏后就不能播放，今天想着再翻翻苹果官方文档，看看能不能找到解决方案。

根据苹果的[File System Programming Guide](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html#//apple_ref/doc/uid/TP40010672-CH2-SW2)，应用打开别的应用的文件，会复制一份到`Documents/Inbox`文件夹中。

具体的描述如下：

> Use this directory to access files that your app was asked to open by outside entities. Specifically, the Mail program places email attachments associated with your app in this directory. Document interaction controllers may also place files in it.
>
> Your app can read and delete files in this directory but cannot create new files or write to existing files. If the user tries to edit a file in this directory, your app must silently move it out of the directory before making any changes. 
>
> The contents of this directory are backed up by iTunes and iCloud.

于是把文件所有的属性都打印出来，发现 `NSFileProtectionKey` 的值为：`NSFileProtectionCompleteUnlessOpen`。 定位到具体的[官方文档](https://developer.apple.com/documentation/foundation/nsfileprotectioncompleteunlessopen)，终于找到了原因:

Files with this type of protection can be created while the device is locked, **but once closed, cannot be opened again until the device is unlocked.** If the file is opened when unlocked, you may continue to access the file normally, even if the user locks the device. There is a small performance penalty when the file is created and opened, though not when being written to or read from. 

原来文件的保护属性为 NSFileProtectionCompleteUnlessOpen时，只有在解锁或者锁屏已经读取的才能读取。那么要改这个问题就简单的，在打开时将文件的保护属性更改掉就可以了，苹果也给出了[示例代码](https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/encrypting_your_app_s_files)：

```swift
do {
   try (fileURL as NSURL).setResourceValue( 
                  URLFileProtection.complete,
                  forKey: .fileProtectionKey)
}
catch {
   // Handle errors.
}

```