---
layout: post
title:  "使用Texture的一些注意事项"
date:   2019-07-16 11:33:10 +0800
tag: Texture
---

### ASTableNode 不支持 separatorInset

[Issue](https://github.com/facebookarchive/AsyncDisplayKit/issues/331)

解决方案：在ASCellNode 中自己增加横线