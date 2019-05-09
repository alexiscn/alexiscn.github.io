---
layout: post
title:  "Cocoa开发 - 对NSTableView进行拖拽排序"
date:   2019-05-05 11:33:10 +0800
tag: 开源项目
---

设置 `draggingDestinationFeedbackStyle`为gap就能实现。

要对

```swift
func tableView(_ tableView: NSTableView, writeRowsWith rowIndexes: IndexSet, to pboard: NSPasteboard) -> Bool {
    let data = NSKeyedArchiver.archivedData(withRootObject: rowIndexes)
    let item = NSPasteboardItem()
    item.setData(data, forType: dragType)
    pboard.writeObjects([item])
    return true
}
```