---
title: Telegram源码-SSignalKit
date: 2022-10-13
tags: [Swift]
top: 402
categories: Swift
---

Telegram-iOS中大规模使用了响应式编程，而SSignalKit则是实现这个响应式编程的核心模块，同时它也是Telegram-iOS项目的基础依赖之一。如果想要阅读学习Telegram-iOS的上层业务代码，它是无法绕过的必经之路。

这套代码的设计略微有点晦涩难懂，主要原因是源代码缺少注释，且没有具体的文档。如果直接从源码阅读，那会因为这部分的代码带着闭包的嵌套，而让阅读者感觉到非常非常绕(有点类似于Promise)，并且会逐渐失去阅读耐心。

为了方便理解，这篇文章还是会从正向的角度来解析SSignalKit，先从宏观的角度来看SSignalKit的设计模式，从而引入SSignalKit各种组件的概念，然后再从源码的角度来逐步分析每一个模块的职责和作用。

## SignalKit的设计

### SignalKit能做什么

### SignalKit设计模式

## SignalKit的组件概念

## SignalKit的组件实现
### Signal

### Subscriber

### Disposable