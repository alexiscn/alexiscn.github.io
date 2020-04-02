---
layout: post
title:  "查看PHAsset的Metadata信息"
date:   2018-04-10 10:21:30 +0800
tag: 开源项目
---

`PHAsset `对象有一些基本属性，可以直接通过他的属性列表获得，比如像素宽高、创建时间、修改时间、以及地理位置信息等。如下所示：

```swift
open var mediaType: PHAssetMediaType { get }

open var mediaSubtypes: PHAssetMediaSubtype { get }

open var pixelWidth: Int { get }

open var pixelHeight: Int { get }

open var creationDate: Date? { get }

open var modificationDate: Date? { get }

open var location: CLLocation? { get }

open var duration: TimeInterval { get }

```

但是仅限于此，假如我们想查看PHAsset对应的EXIF信息的话，需要请求他的编辑信息，拿到照片的URL，然后通过`CoreImage`里面的CIImage相关API获得更加详细的信息，诸如拍摄设备是什么，光圈等。

```swift
let asset: PHAsset = ...
asset.requestContentEditingInput(with: nil) { (input, _) in
    if let url = input?.fullSizeImageURL, let image = CIImage(contentsOf: url) {
        // TODO
    } else {
        self.showError("can not dig Photo...")
    }
}
```

定义如下的结构，用于TableView中展示metadata信息

```swift
typealias MetadataInfo = (key: String, value: Any)

struct MetadataGroup {
    let title: String
    let metadatas: [MetadataInfo]
}
```

然后对CIImage增加一个扩展方法，将其`properties`转成MetadataGroup

```swift
extension CIImage {

    func metadataGroups() -> [MetadataGroup] {
        let metadata = properties.sorted(by: { $0.key < $1.key })
        var groups: [MetadataGroup] = []
        var basics: [MetadataInfo] = []
        metadata.forEach({ (key, value) in
            if let dict = value as? [String: Any] {
                let title = key.replacingOccurrences(of: "{", with: "").replacingOccurrences(of: "}", with: "")
                groups.append(MetadataGroup(title: title, metadatas: dict.sorted(by: { $0.key < $1.key })))
            } else {
                basics.append((key, value))
            }
        })
        groups.insert(MetadataGroup(title: "BASIC", metadatas: basics), at: 0)
        return groups
    }
}
```

代码在 [github](https://github.com/alexiscn/PhotoDigger)上能找到。