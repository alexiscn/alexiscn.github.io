---
layout: post
title:  "使用markjs渲染markdown文件"
date:   2021-04-29 10:02:10 +0800
tag: ios
---

[Marked](https://marked.js.org/)是用来将Markdown文件渲染为HTML的JavaScript库。用法也非常简单，使用CLI命令：

```bash
marked -s "*hello world*"
```

可以得到渲染结果：

```html
<p><em>hello world</em></p>
```

同时Marked还能在浏览器中使用，只需要引入`marked.min.js`，调用 `marked()`即可。

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>Marked in the browser</title>
</head>
<body>
  <div id="content"></div>
  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <script>
    document.getElementById('content').innerHTML =
      marked('# Marked in browser\n\nRendered by **marked**.');
  </script>
</body>
</html>
```

这样我们就可以利用`WKWebView`与`Marked`来渲染Markdown文件了。使用在浏览器中渲染的方法，我们需要定义一套HTML的模板，用来渲染Markdown。

```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width,initial-sacle=1.0,minimum-scale=1.0,maximum-scale=1.0,user-scrollable=no"/>
    <link rel="stylesheet" href="default-light.css">
    <link rel="stylesheet" href="default-dark.css" media="(prefers-color-scheme: dark)">
    <script src="marked.min.js"></script>
    <style>
        .markdown-body {
            box-sizing: border-box;
            min-width: 200px;
            max-width: 980px;
            margin: 0 auto;
            padding: 45px;
        }

        @media (max-width: 767px) {
            .markdown-body {
                padding: 15px;
            }
        }
    </style>
</head>
<body>
    <article class="markdown-body" id="content"></article>
    <script>
        function renderHTML(encoded) {
            const html = _decodeBase6(encoded);
            document.getElementById('content').innerHTML = marked(html);
        }
    
        function _decodeBase6(str) {
            /// https://developer.mozilla.org/zh-CN/docs/Glossary/Base64
            return decodeURIComponent(atob(str).split('').map(function(c) {
                return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
            }).join(''));
        }
    </script>
</body>
</html>

```

在使用的时候有几个需要注意的地方:

- 传入的Markdown文本需要经过URLEncode，因为我们之间调用方法的时候是传递的字符串，不转义会导致函数调用失败
- 默认Marked不提供高亮，我们需要自己集成 [markdown.css](https://github.com/sindresorhus/github-markdown-css)
- 如果需要适配DarkMode，我们还需要实现DarkMode下的 markdown.css
- 需要将`WKWebView`的`isOpaque`设置`false`，以适配DarkMode

大致的目录结构如下：

![](/assets/images/2021/markedjs-dir.png)

原生端代码也十分简单，就只贴关键代码了：

```swift
    private func setup() {
        // .....
        let url = EditorBundle.url(forResource: "Render.html")!
        let html = try! String(contentsOf: url)
        webView.loadHTMLString(html, baseURL: url.deletingPathExtension())        
    }

    private func render(content: String) {
        let data = content.data(using: .utf8) ?? Data()
        let script = "renderHTML('\(data.base64EncodedString())')"
        webView.evaluateJavaScript(script) { (result, error) in
            if let error = error {
                print(error)
            }
        }
    }
```