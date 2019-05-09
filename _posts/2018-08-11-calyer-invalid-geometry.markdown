---
layout: post
title:  "CALayerInvalidGeometry 异常"
date:   2018-08-11 10:21:10 +0800
tag: iOS杂货铺
---

在项目中，偶尔会遇到 CALayerInvalidGeometry 的错误。


```
CALayerInvalidGeometry CALayer position contains NaN: [160 nan
```

一般的原因是 Frame的计算出现了 NaN，即出现了除以0的情况