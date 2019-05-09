---
layout: post
title:  "SQLiteKit for iOS"
date:   2018-06-15 10:21:10 +0800
tag: 开源项目
---

最近在做的项目中涉及到数据库，在iOS中使用数据库。

于是就参考 [SQLite-Net](https://github.com/praeclarum/sqlite-net/) 的方式重复造了一个轮子，在造轮子的时候也遇到一些问题记录一下。


使用方法：

* 定义你的数据库模型，实现 `SQLiteTable`协议
* 使用`SQLiteConnection`建立数据库连接
* 在`SQLiteConnection`上进行数据库的操作，如 `insert`、`update`、`delete`
* 使用`SQLiteTableQuery`进行数据库的查询操作，如`toList` 是查看所有列表。


1、实现原理

写入数据

读取数据，利用了`Codable`的特性，将从数据库中读取的数据转换为我们想要的model.

2、遇到的挑战

插入某一条记录时，假如主键是自增类型的，需要把自增的结果修改到插入的模型中。由于`SQLiteTable`继承的是`class`，并不是NSObject，所以无法使用`setValue(forKey:)`的方式来给主键重新赋值。

1、Any? 的值为nil时，value == nil 为false


2、CodingKey 的问题

使用 `Mirror`能反射拿到类型的基本信息，如变量名称，变量类型。

```swift

struct User {
    let name: String
    let avatar: Data?
}

let user = User(name: "Alex", avatar: nil)

let m = Mirror(reflecting: user)
m.children.forEach { child in
    print("key:\(child.label), value type:\(type(of: child.value))")
}

// 输出：
// key:Optional("name"), value type:String
// key:Optional("avatar"), value type:Optional<Data>
```

Swift 4.2 增加遍历Enum所有值的API, `CaseIterable`。对于 Swift4.2 以下也可以实现`CaseIterable`：

```swift
public protocol EnumCaseIterable {
    associatedtype AllCases: Collection where AllCases.Element == Self
    static var allCases: AllCases { get }
}

extension EnumCaseIterable where Self: Hashable {
    
    static var allCases: [Self] {
        return [Self](AnySequence { () -> AnyIterator<Self> in
            var raw = 0
            var first: Self?
            return AnyIterator {
                let current = withUnsafeBytes(of: &raw) { $0.load(as: Self.self) }
                if raw == 0 {
                    first = current
                } else if current == first {
                    return nil
                }
                raw += 1
                return current
            }
        })
    }
}
```

对于普通的Enum是可以正常拿到`Key`，以及`rawValue`对应的值：

```swift
enum Fruit: String, EnumCaseIterable {
    case apple = "Big Apple"
    case pear = "pear"
}

for f in Fruit.allCases {
    print(f) // 输出 apple、pear
    print(f.rawValue) // 输出 Big Apple、pear
}
```

然鹅对于 CodingKey，因为 CodingKey实现了 `CustomDebugStringConvertible`, `CustomStringConvertible`

```swift
enum Car: String, EnumCaseIterable, CodingKey {
    case tesla = "TTT"
    case bmw
}

for car in Car.allCases {
    print(car) // 输出 Car(stringValue: "TTT", intValue: nil) 、Car(stringValue: "bmw", intValue: nil)
    print(car.rawValue) // 输出 TTT、bmw
}
```

