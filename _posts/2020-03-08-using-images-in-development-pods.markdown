---
layout: post
title:  "Image Catalog in DevelopmentPods"
date:   2020-03-08 19:02:10 +0800
tag: tips
---


[CocoaPods](https://cocoapods.org/) supports using `xcassets` in your pod library. Add `resource_bundle` to your `podspec`:

```
s.resource_bundle = { 'Media' => 'Sources/Media.xcassets'}
```

And here is your diretory looks like:

```
│── Sources
│   ├── Info.plist
│   ├── Media.xcassets
│   │   ├── ChatRoom_Bubble_Text_Receiver_White_57x40_.imageset
│   │   │   ├── ChatRoom_Bubble_Text_Receiver_White_57x40_@2x.png
│   │   │   ├── ChatRoom_Bubble_Text_Receiver_White_57x40_@3x.png
│   │   │   └── Contents.json
│   │   └── ChatRoom_Bubble_Text_Sender_Green_57x40_.imageset
│   │       ├── ChatRoom_Bubble_Text_Sender_Green_57x40_@2x.png
│   │       ├── ChatRoom_Bubble_Text_Sender_Green_57x40_@3x.png
│   │       └── Contents.json
```

Create a helper function to simplify loading images:

```swift
class Utility {
    
    static var bundle: Bundle? = {
        if let url = Bundle(for: WXUtility.self).url(forResource: "Media", withExtension: "bundle") {
            return Bundle(url: url)
        }
        return nil
    }()
    
    static func image(named: String) -> UIImage? {
        return UIImage(named: named, in: bundle, compatibleWith: nil)
    }
}
```

Now you can load your image using following code:

```swift
let image = Utility.image(named: "my_img")
```