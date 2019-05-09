---
layout: post
title:  "用原生方式查看WKWebView的图片"
date:   2019-01-02 21:23:15 +0800
categories: Swift iOS WKWebView
tag: iOS杂货铺
---

用原生的方式在WKWebView查看图片的实现方式有多种，可以使用JavaScript注入，也可以用 WKUserController增加handler的方式。JavaScript注入相对来说比较简单些。

原理： JavaScript端遍历所有的图片，重写点击事件，点击图片时设置特殊scheme的跳转。原生端捕获URL跳转事件，判断特殊的scheme，解析URL，用原生的方式展示图片。

```javascript
var images = document.getElementsByTagName('img');
for (var i = 0; i < images.length; i ++) {
    var img = images[i];
    var src = img.src;
    img.onclick = (function(urlString) {
        return function() {
            var encodedURI = encodeURIComponent(urlString)
            location.href = "nativeProtocol://" + encodedURI
        }
    })(src)
}
```

需要注意的是需要将图片的URL进行URL Encode。

由于JavaScript需要在DOM加载完成后才有效果，所有就在WKWebView的 `didFinish navigation`中注入一次就好。

```swift
func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
    let jsCode = ... // read from file or in code
    webView.evaluateJavaScript(jsCode, completionHandler: nil)
}
```

然后点击图片会出发如下的回调。

```swift
func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
    if let url = navigationAction.request.url, url.scheme == "nativeProtocol" {
            let src = url.absoluteString.replacingOccurrences(of: "nativeProtocol://", with: "")
            if let url = src.removingPercentEncoding {
                // get image url
            }
    }
}
```

获得图片的URL就可以用原生方式在展示了。还可以做得更多，比较点击图片后的transition，就需要获得图片的点击区域，以及图片大小。