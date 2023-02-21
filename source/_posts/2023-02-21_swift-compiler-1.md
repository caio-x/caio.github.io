---
title: Swift编译原理(1)
date: 2023-02-21
tags: [Swift]
top: 401
categories: Swift
---

深入理解Swift编译原理 -- Swift源码编译

<!-- more -->

2023年第一个OKR的flag就是看完swift-frontend部分的代码。

工欲善其事必先利其器，学Swift不可缺少的就是编译swift代码，最近折腾了几天，终于能够编译通过了。

### 环境
1. 操作系统：MacOS Monterey 12.5, 至少100G的空间
2. cmake: 3.24.2
3. python: 3.7.7
4. Swift：branch release/5.7

之前试过通过Ninja编译的代码，在VSCode下的调试还是不如XCode下丝滑，这里还是建议使用XCode。

### 编译步骤

基本步骤可以按照[官方的文档](https://github.com/apple/swift/blob/main/docs/HowToGuides/GettingStarted.md)操作。

1. create directory
```
    mkdir swift-project
    cd swift-project
```

2. clone code
```
    git clone https://github.com/apple/swift.git swift
    cd swift
    utils/update-checkout --clone
```

3. clone dependencies
```
utils/update-checkout --scheme ${mybranchname}
# OR
utils/update-checkout --tag ${mytagname}
```
如果是全新clone编译，这步会在swift源码文件夹的平级目录下，clone下来当前swift源码分支所依赖的其他模块代码。

4. actual build
这一步和官方文档的描述有点出入，搞了很久才找到的解决方案(因为编译一次实在要很久很久)

最终成功的脚本 `./utils/build-script --debug --xcode --bootstrapping=off`，这个过程会很久。

> `--release-debuginfo` 和 `--debug`不能同时存在。而且需要加上`--bootstrapping=off`参数，否则都会编译不通过。前者可以[参考这个](https://forums.swift.org/t/problems-with-build-script-building-compiler-with-xcode/53477/6)

### 调试编译器
在编译完成后，在`./build/Xcode-DebugAssert/swift-macosx-x86_64/`文件夹(m1的为arm)下会生成xcodeproject文件.

打开工程文件后，点击自动生成，会生成很多个scheme。找到ALL_BUILD，点击编译。

编译通过后查看工程的产物:
![Xnip2023-02-21_19-02-21](Xnip2023-02-21_19-02-21.jpg)

发现swift、swiftc可执行文件都是一个软链接，链接均指向了swift-frontend这个可执行文件。
![Xnip2023-02-21_19-09-04](Xnip2023-02-21_19-09-04.jpg)

在driver.cpp内的 `run_driver` 函数中可以看到swift,swiftc,swift-frontend 其实就是是同一个文件。
![Xnip2023-02-21_19-16-57](Xnip2023-02-21_19-16-57.jpg)

通过ExecName来指定一个"--driver-mode", 关于这个后面文章会再细讲。

现在已经确定了，要调试的就是`swift-frontend`。接下来，找到`swift-frontend`这个**可执行文件**的编译scheme。将编译命令填充到启动参数内，这里简单将第一个参数填`-emit-sil`, 即编译swift代码生成优化后的sil。第二个参数就是源码文件路径。

![Xnip2023-02-21_19-20-18](Xnip2023-02-21_19-20-18.jpg)

之后随便找个函数打个断点，Run code，可以「愉快地」调试了。
![Xnip2023-02-21_19-28-10](Xnip2023-02-21_19-28-10.jpg)

