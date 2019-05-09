---
layout: post
title:  "基于Alamofire与Codable的网络封装"
date:   2018-01-25 10:43:10 +0800
categories: Swift Codable
tag: 开源项目
---

# 一、原理

在实际iOS项目开发中，必然会涉及到写一个API访问，我们直接使用Alamofire进行网络请求：

```swift
Alamofire.request("https://api.github.com/gists").responseJSON { (response) in
    // do something with response.result
}
```

当收到服务端返回的数据后，对response.result进行处理，result是Any类型，我们需要将result转换成我们想要的类型。

Swift 4中推出了 Codable，让序列化JSON变得更加简单：

```swift
do {
  let jsonObject = try JSONDecoder().decode(T.self, from: data)
} catch let error as NSError {
  print(error)
}
```

我们可以拿网络返回的data与定义的类型，去将data序列化为我们想要的model。然而，如果每个API都需要这样做，显得比较麻烦。

庆幸的是，Swift中提供了泛型，我们可以结合泛型和Codable，实现一个通用的框架：输入一个类型，输出该类型的实例，如下图所示：

![](/assets/images/generic-networking-flow.jpg)

抽空基于Codable和Generic封装了一个网络请求框架：[GenericNetworking](https://github.com/alexiscn/GenericNetworking)

# 二、用法

前阵子对比特币比较感兴趣，又看到火币网提供了[API接口](https://github.com/huobiapi/API_Docs/wiki/REST_introduction)，于是基于GenericNetworking封装了火币网的API: [HuobiSwift](https://github.com/alexiscn/HuobiMac/tree/master/DevelopmentPods/HuobiSwift) ， 就用这个来演示如何在项目中使用GenericNetworking。

## 1. 定义你的API SDK类，如 HuobiAPI.swift

```swift
public class HuobiAPI {
    fileprivate static let baseURLString = "https://api.huobi.pro"
}
```

根据不同的主path，可以定义不同的扩展类来实现API接口。

## 2. 配置GenericNetworking

GenericNetworking 支持配置baseURLString，即你API的host路径。这样之后访问接口的时候只要使用path加参数就可以了。

```swift
GenericNetworking.baseURLString = "your_base_url_string"
```

GenericNetworking 还支持配置默认的HTTP头，会在最后请求的时候作为请求header。

```swift
GenericNetworking.defaultHeaders = ["HTTP_HEADER": "HTTP_HEADER_VALUE"]
```


## 3. 定义服务端返回的Model

按照API返回的json，定义对应的模型，一般用实现Codable协议的struct定义即可，如果有一些字段服务端可能没有返回，可以定义为可选类型（optional）

```swift
public struct HBSupportSymbols: Codable {
    public let status: String
    public let data: [HBSupportSymbol]
}

public struct HBSupportSymbol: Codable {
    
    enum CodingKeys: String, CodingKey {
        case baseCurrency = "base-currency"
        case quoteCurrency = "quote-currency"
        case pricePrecision = "price-precision"
        case amountPrecision = "amount-precision"
        case symbolPartition = "symbol-partition"
    }
    
    /// 基础币种
    public let baseCurrency: String
    /// 计价币种
    public let quoteCurrency: String
    /// 价格精度位数（0为个位）
    public let pricePrecision: String
    /// 数量精度位数（0为个位）
    public let amountPrecision: String
    /// 交易区,main主区，innovation创新区，bifurcation分叉区
    public let symbolPartition: String
}
```

## 4. 定义具体API的方法

最后定义API的方法即可，GenericNetworking请求返回的是一个DataRequest对象，我们可以选择忽略请求返回结果，也可以对其进行处理。

```swift 
// MARK: - Common API
extension HuobiAPI {
    
    ///  查询系统支持的所有交易对及精度
    ///
    /// - Parameter completion: 请求回调
    public class func getSymbols(completion: @escaping GenericNetworkingCompletion<HBSupportSymbols>) {
        let path = "/v1/common/symbols"
        GenericNetworking.getJSON(path: path, completion: completion)
    }
}
```

这样在业务逻辑层，只需要调用HuobiAPI 的 getSymbols方法就可以了。

```swift
HuobiAPI.getSymbols { (response) in
    switch response {
    case .error(let error):
        print(error)
    case .success(let symbols):
        // do something with symbols
        // symbols is typeof HBSupportSymbols
        print(symbols.status)
    }
}
```

# 三、后续

目前GenericNetworking只实现了 GET、POST的方法，并且不支持下载、上传。如果后面实际开发项目有需要再加上。当然你也可以fork后修改....