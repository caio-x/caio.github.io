---
title: Swift - 踩坑集锦
date: 2022-08-24
tags: [Swift]
top: 401
categories: Swift
---

记录使用Swift的时候遇到的一些奇葩问题。

<!-- more -->

## Swift 踩坑集锦

### Optional 的推导问题
```swift
var list:[String]? = ["X", "Y"]

var dic:[String : Any] = [:]
dic["Z"] = (list ?? []).map {
    ["V" : $0]
}

let value = (list ?? []).map {
    ["V" : $0]
}
print(dic)
print(value)
```

上述代码中`dic`的`Z`内存储的值，和`value`虽然是同一个表达式的计算结果，但是最终的值是不一样的。

`Z`最终的值为`["V": ["X", "Y"]]`, value为`[["V": "X"], ["V": "Y"]]`

> 问题根本原因

在Swift的推导系统中，`Z`的右值被推导为`Optional.map()`, value的右值被推导为`Array.map()`。

这个是`Collection`的map源码
```swift
@inlinable
  public func map<T>(
    _ transform: (Element) throws -> T
  ) rethrows -> [T] {
    // TODO: swift-3-indexing-model - review the following
    let n = self.count
    if n == 0 {
      return []
    }

    var result = ContiguousArray<T>()
    result.reserveCapacity(n)

    var i = self.startIndex

    for _ in 0..<n {
      result.append(try transform(self[i]))
      formIndex(after: &i)
    }

    _expectEnd(of: self, is: i)
    return Array(result)
  }
```

这个是`Optional`的map源码
```swift
@inlinable
  public func map<U>(
    _ transform: (Wrapped) throws -> U
  ) rethrows -> U? {
    switch self {
    case .some(let y):
      return .some(try transform(y))
    case .none:
      return .none
    }
  }
```

可以看到这两个map的函数原型都不一样，`Collection.map`是返回泛型数组，而`Optional.map`是返回泛型值。

`Collection.map`的作用是，对每一个Collection内的元素执行`transform`闭包，将`transform`返回的每一个元素merge为最终的数组返回，这个是熟知的Array的map操作。

`Optional.map`的作用更多的是屏蔽掉`Optional`对于运算符的影响，避免更多解包的胶水代码出现，官方文档举了个例子:

```swift
    let possibleNumber: Int? = Int("42")
    let possibleSquare = possibleNumber.map { $0 * $0 }
    print(possibleSquare)
    // Prints "Optional(1764)"

    let noNumber: Int? = nil
    let noSquare = noNumber.map { $0 * $0 }
    print(noSquare)
    // Prints "nil"
```
`Optional.map`可以避免`Optional`类型没有重载`*`运算符而导致的编译问题。

回过头来分析`dic`中`Z`的右值，如果`list ?? []`被推导为`Optional`,那么`map`闭包中的`$0`就是`Optional`内的非空值，结果确实就是`["V": ["X", "Y"]]`。

> 为什么会出现这个不一致的推导结果？

如果把测试代码中list解包改成这样:
```swift
let list_1 = list ?? []
dic["Z"] = list_1.map {
    ["V" : $0]
}
```

或者在定义字典的时候不使用`Any`, 而是直接明确类型:
```swift
var dic:[String : [[String : String]]] = [:]
```

`dic`中`Z`的值和`value`就保持一致了。

`Optional`重载了两个"??"运算符，这两个区别在于"??"右参是否依旧为`Optional`类型的值。上面的测试代码中右参为`[]`, 很明显是一个数组类型。

这里先推测一下，在左值为`Any`类型的时候，右值当出现`??`解包的时候，类型推导系统可能会出错。