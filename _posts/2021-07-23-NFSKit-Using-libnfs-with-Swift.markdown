---
layout: post
title:  "NFSKit: 在Swift中使用libnfs"
date:   2021-07-23 14:03:10 +0800
tag: ios
---

最近在给Filebox添加了nfs的功能，苦于没有现成的轮子可用，于是就参考 [AMSMB2](https://github.com/amosavian/AMSMB2) 自己撸了一个库: [NFSKit](https://github.com/alexiscn/NFSKit)。


由于尝试使用脚本将libnfs打包为libnfs.a失败，索性就直接复制源代码做成一个nfs的Swift Package. 然后在Package 中定义一下宏，当然可能有一些重复的宏定义，等有时间的可以优化一下。
甚至可以写一个bash脚本，自动从libnfs 拉取代码，复制到Sources目录下。


```swift

import PackageDescription

let package = Package(
    name: "nfs",
    products: [
        // Products define the executables and libraries a package produces, and make them visible to other packages.
        .library(
            name: "nfs",
            targets: ["nfs"]),
    ],
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        // .package(url: /* package url */, from: "1.0.0"),
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages this package depends on.
        .target(
            name: "nfs",
            dependencies: [], cSettings: [
                .headerSearchPath("include/nfsc"),
                .headerSearchPath("include"),
                .headerSearchPath("mount"),
                .headerSearchPath("nfs"),
                .headerSearchPath("nfs4"),
                .headerSearchPath("nlm"),
                .headerSearchPath("portmap"),
                .define("HAVE_CONFIG_H", to: "1"),
                .define("_U_", to: "__attribute__((unused))"),
                .define("HAVE_GETPWNAM", to: "1"),
                .define("HAVE_SOCKADDR_LEN", to: "1"),
                .define("HAVE_SOCKADDR_STORAGE", to: "1"),
                .define("HAVE_TALLOC_TEVENT", to: "1")
            ]),
        .testTarget(
            name: "nfsTests",
            dependencies: ["nfs"]),
    ]
)
```

由于nfs 中全是c代码，在实际项目中使用的时候肯定不方便，需要给nfs再包装一层：NFSKit。代码很多直接从 AMSMB2 复制而来，就不多说了，直接看源代码。


```swift
import NFSKit

class NFSBrowser {
    
    var client: NFSClient?
    
    // url: nfs://xxx.xxx.xxx.xxx
    init?(url: URL) throws {
        client = try NFSClient(url: url)
    }
    
    func listExports(handler: @escaping (Result<[String], Error>) -> Void) {
        client?.listExports(completionHandler: handler)
    }
    
    func mount(export: String, completion: @escaping (Result<Void, Error>) -> Void) {
        client?.connect(export: export) { error in
            DispatchQueue.main.async {
                if let error = error {
                    completion(.failure(error))
                } else {
                    completion(.success(()))
                }
            }
        }
    }

    func listDirectory(at path: String) {
        client?.contentsOfDirectory(atPath: path) { result in
            switch result {
            case .success(let items):
                for entry in items {
                    print("name:", entry[.nameKey] as! String,
                          ", path:", entry[.pathKey] as! String,
                          ", type:", entry[.fileResourceTypeKey] as! URLFileResourceType,
                          ", size:", entry[.fileSizeKey] as! Int64,
                          ", modified:", entry[.contentModificationDateKey] as! Date,
                          ", created:", entry[.creationDateKey] as! Date)
                }
            case .failure(let error):
                print(error)
            }
        }
    }

    func moveItem(atPath path: String, to toPath: String) {
        client?.moveItem(atPath: path, toPath: toPath) { error in
            if let error = error {
                print(error)
            }
        }
    }

    func removeItem(atPath path: String) {
        client?.removeItem(atPath: path) { error in
            if let error = error {
                print(error)
            }
        }
    }
    
    func downloadItem(atPath path: String) {
        let filePath = NSTemporaryDirectory().appending("temp.data")
        let fileURL = URL(fileURLWithPath: filePath)
        let progress = Progress(totalUnitCount: 0)
        client?.downloadItem(atPath: path, to: fileURL) { bytes, total in
            progress.totalUnitCount = total
            progress.completedUnitCount = bytes
            print(progress.fractionCompleted)
            return true
        } completionHandler: { error in
            if let error = error {
                print(error)
            }
        }
    }
    
    func upload(data: Data, toPath: String) {
        let progress = Progress(totalUnitCount: Int64(data.count))
        client?.write(data: data, toPath: toPath) { (uploaded) -> Bool in
            progress.completedUnitCount = uploaded
            print(progress.fractionCompleted)
            return true
        } completionHandler: { error in
            if let error = error {
                print(error)
            }
        }
    }
}
```