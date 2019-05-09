---
layout: post
title:  "在Swift私有Pod中使用CommonCrypto"
date:   2018-01-29 17:42:10 +0800
categories: Swift CommonCrypto Pod
tag: iOS杂货铺
---

由于CommonCrypto 不是一个独立的模块，在Swift中不能直接 import 来使用。在Swift主项目中使用 CommonCrypto比较简单，只需要在Bridge Header 中添加引用就可以了。


在Pod中使用CommonCrypto稍微有点复杂，稍微记录下，假设我们的私有Pod名字叫: `CommonKit`

## 1、在`CommonKit` 根目录下创建 RunScript.sh文件，将如下的文件复制到RunScript.sh文件中

```sh
#!/bin/sh
COMMOM_CRYPTO_PATH=$SDKROOT/usr/include/CommonCrypto/CommonCrypto.h'
COMMOM_CRYPTO_R_PATH=$SDKROOT/usr/include/CommonCrypto/CommonRandom.h

MODULE_DIR="$SRCROOT/Modules/CommonCrypto"
MODULE_FILE=$MODULE_DIR/module.map
MODULE_TEMPLATE="module CommonCrypto [system] {\n\t
header \"$COMMOM_CRYPTO_PATH\"\n\t
header \"$COMMOM_CRYPTO_R_PATH\"\n\t
export *\n
}"

echo "Create Modules path to map CommonCrypto lib"
mkdir -p "$SRCROOT/Modules/CommonCrypto"

echo "Cleanup previous CommonCrypto script to make sure the deployment target is always updated"

echo "" > $MODULE_FILE

echo "Create CommonCrypto module map template"
echo -e $MODULE_TEMPLATE > $MODULE_FILE'}

```

## 2、然后修改 `CommonKit.podsepc` 

```sh
s.script_phase = { :name => '', :script => 'sh ${PODS_TARGET_SRCROOT}/RunScript.sh' }
s.xcconfig = { 'SWIFT_INCLUDE_PATHS' => '$SRCROOT/Modules' }
```

## 3、运行 `pod install`

会执行 RunScript.sh中的脚本，会在Pods目录下创建Modules/CommonCrypto目录，该目录下包含一个 module.map 文件

这样在私有Pod中就可以直接 `import CommonCrypto` 来使用CommonCrypto中的类了。