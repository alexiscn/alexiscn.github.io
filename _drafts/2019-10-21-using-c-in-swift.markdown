---
layout: post
title:  "Swift调用C"
date:   2019-10-21 11:33:10 +0800
tag: Swift
---

类型对应表

| C语法  |  Swift语法 |
| -- | -- |
| const Type * | UnsafePointer<Type> |
| Type * | UnsafeMutablePointer<Type> |
| Type * const * | UnsafePointer<Type> |
| Type * __strong * | UnsafeMutablePointer<Type> |
| Type ** | AutoreleasingUnsafeMutablePointer<Type> |
| const void * | UnsafeRawPointer |
| void * | UnsafeMutableRawPointer |

### 处理回调函数

有时候Swift调用的C函数有回调函数作为参数。比如如下的C函数：

```c
/*
 * Callback for all ASYNC rpc functions
 */
typedef void (*rpc_cb)(struct rpc_context *rpc, int status, void *data,
                       void *private_data);

EXTERN int mount_getexports_async(struct rpc_context *rpc, const char *server,
                                  rpc_cb cb, void *private_data);
```

有两种方式，一种是定义一个全局的函数，类型与callback中的类型保持一致，一种是用block的方式。

```swift
// 方式一
class Test1 {
    func callCFuntionWithCallbackMethod1() {
        let result = mount_getexports_async(rpc, "192.168.1.2", (context, status, data, private_date) -> Void {

        }, cbdata)
        print(result)
    }
}


// 方式二:

class Test2 {
    func callCFuntionWithCallbackMethod2() {
        let result = mount_getexports_async(rpc, "192.168.1.2"), getexportscalback, cbdata)
        print(result)
    }
}

func getexportscalback() {

}
```

参考：

- [Using Imported C Functions in Swift](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/using_imported_c_functions_in_swift)