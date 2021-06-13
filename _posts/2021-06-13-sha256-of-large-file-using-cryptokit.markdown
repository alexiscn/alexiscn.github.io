---
layout: post
title:  "使用CryptoKit计算大文件的SHA256"
date:   2021-06-13 10:02:10 +0800
tag: ios
---

某一些时候我们需要计算大文件的SHA256，如果将data全部读取出来进行计算，很有可能因为内存不足而crash，所以我们需要使用其他的方法进行处理，苹果在其官方文档中已经给出了说明。

> You can compute the digest by calling the static hash(data:) method once. Alternatively, if the data that you want to hash is too large to fit in memory, you can compute the digest iteratively by creating a new hash instance, calling the update(data:) method repeatedly with blocks of data, and then calling the finalize() method to get the result.

我们使用 FileHandle 分批读取大文件的数据，再丢给SHA256进行处理，代码很简单如下：

```swift
import CryptoKit

do {
    let fileHandle = try FileHandle(forReadingFrom: fileURL)
    let bufferSize = 1024 * 1024
    var sha256 = SHA256()

    var loop = true
    while loop {
        autoreleasepool {
            let data = fileHandle.readData(ofLength: bufferSize)
            if data.count > 0 {
                sha256.update(data: data)
            } else {
                loop = false
            }
        }
    }
    let sha256Hash = sha256.finalize()
} catch {
    print(error)
}


```