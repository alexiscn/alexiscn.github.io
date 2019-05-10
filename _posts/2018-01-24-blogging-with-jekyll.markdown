---
layout: post
title:  "用Jekyll搭建博客"
date:   2018-01-24 16:28:10 +0800
categories: Jekyll Github
---

用Github Pages 配合 Jekyll 写博客十分方便，只需要一些简单的配置就能完成了，而且也不需要自己购买VPS作为托管，并且还可以自定义域名。

## 当前环境 

* macOS 
* Terminal

## 本机安装Jekyll

安装Jekyll十分简单，可以运行如下的命令。可能需要注意的是ruby的版本不能太旧，否则会报一堆错误。

```bash
$ gem install bundler jekyll
```

## 创建博客与预览

通过 `jekyll new 博客名称`来创建一个新的博客

生成的目录文件如下：

<p style='text-align:center'>
<img src='/assets/images/2018/jekyll-blog-files.png' width='423' />
</p>

稍微解释一下文件夹的内容：

* _config.yml 是用来配置
* _posts 文件夹主要存放markdown格式的文章
* 404.html 当访问一个地址找不到的时候显示404.html
* about.md
* Gemfile
* index.md


运行 `bundle exec jekyll serve` 进行本地预览，效果图如下：

<p style='text-align:center'>
<img src='/assets/images/2018/jekyll-blog-preview.jpg' width='500'/>
</p>

## 配置主题

通过 `jekyll new`命令创建的博客默认使用的`minima`主题。可以配置如标题

## 配置域名

在Github网页上配置域名

## 配置 favicon

编辑 `head.html`文件，在

```html
<link rel="stylesheet" href="{{ "/assets/main.css" | relative_url }}">
```

下插入如下的代码：

```html
<link rel="shortcut icon" type="image/png" href="/favicon.png">
```

favicon.png 放在根目录下，或者也可以放在 assets目录下，如果放在 assets 目录下，对应的代码就要修改为

```html
<link rel="shortcut icon" type="image/png" href="/assets/favicon.png">
```

## 本地草稿

可以在目录下增加 `_drafts` 文件夹


## 预览博客

运行命令 `bundle exec jekyll serve`，

如果运行成功，在浏览器里面输入 `http//127.0.0.1:4000`就能预览博客了。