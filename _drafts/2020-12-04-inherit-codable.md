---
layout: post
title:  "Swift Codable 继承"
date:   2020-12-04 15:30:10 +0800
tag: iOS
---

在Swift中Codable

```json
{
    "name": "Tom",
    "age": 20
}
```

对应Swift

```swift
struct Student: Coable {

    let age: Int

    let name: String

}

do {
    let student = try JSONDecoder().decode(data)
} catch {
    print(error)
}

```

```swift
struct Student: Coable {

    let age: Int

    let name: String

    
}

do {
    let student = try JSONDecoder().decode(data)
} catch {
    print(error)
}

```