---
layout: post
title:  "Mapbox截图包含路线路"
date:   2018-09-04 20:12:10 +0800
categories: Swift iOS Mapbox
---

Mapbox 是一款不错的地图提供商，有多种样式可选，也可以编辑样式。

Mapbox 本身提供了对地图的截图方法。

```swift
let options = MGLMapSnapshotOptions(styleURL: mapView.styleURL, camera: mapView.camera, size: rect.size)
options.zoomLevel = mapView.zoomLevel
var snapshotter: MGLMapSnapshotter? = MGLMapSnapshotter(options: options)
snapshotter?.start { (snapshot, error) in
    if let error = error {
    print(error)
    }
    completion(snapshot?.image)
    snapshotter = nil
}
```

但是这个方法比较慢，而且只能对地图进行截图，不能包含地图中的标记、路线图。所以只能采取对屏幕进行截图，然后进行区域裁切。

```swift
UIGraphicsBeginImageContextWithOptions(view.bounds.size, false, 0.0)
drawHierarchy(in: view.bounds, afterScreenUpdates: true)
let screenshot = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()

guard let image = screenshot else {
    completion(nil)
    return
}
let scale = image.scale
let cropRect = CGRect(x: rect.origin.x * scale, y: rect.origin.y * scale, width: rect.width * scale, height: rect.height * scale)
guard let cgImage = image.cgImage?.cropping(to: cropRect) else {
    completion(nil)
    return
}
let result = UIImage(cgImage: cgImage, scale: image.scale, orientation: image.imageOrientation)
completion(result)
```

其中 view 是包含 mapView的容器。