---
layout: post
title:  "自定义UISearchBar"
date:   2019-08-19 11:33:10 +0800
tag: 自定义UISearchBar
---


### 设置后tableHeaderView为UISearchBar，在下拉的情况下会露出来一块背景颜色后，下拉列表会出现默认背景色


将UITableView的tableHeaderView 设置为 UISearchBar，在下拉的情况下会露出来一块背景颜色


<img src='/assets/images/2019/UISearchBar_UITableViewHeader_Background@2x.png' width='414' />

可以看到明显UITableView背景色、导航栏的背景色与下拉UISearchBar的背景色明显不一样。查看UI视图的时候，发现UITableView中多了一个View，这个View就是颜色不一样的视图。

解决方案是将 tableView的 backgroundView 设置为一个空的View。


```swift
tableView.backgroundView = UIView()
```

### 移除UISearchBar底部的黑线

<img src='/assets/images/2019/UISearchBar_BotomLine@2x.jpg' width='433' />

搜了下网络上的一些方案，将 `layer.borderColor`设为背景色，但是在切换后还是会出现黑线。


```swift
searchViewController?.searchBar.setBackgroundImage(UIImage(), for: .any, barMetrics: .default)
searchViewController?.searchBar.setBackgroundImage(UIImage(), for: .any, barMetrics: .defaultPrompt)

```

<img src='/assets/images/2019/UISearchBar_RemovedLine@2x.jpg' width='438' />



