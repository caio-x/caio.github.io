---
title: 理解C++内存排序和可见性
date: 2022-07-20
tags: [C++]
top: 500
categories: C++
---

本文主要讨论了C++在原子操作上的内存模型以及内存排序标准。

**相对**来说，C++已经是比较接近系统底层的语言了，C++设计也是希望如此，毕竟它本身已经如此复杂了。在C++ 11的标准里，推出了非常多的新特性，它们让C++变得非常灵活，而它们都依赖于标准里的多线程内存模型。但可能大部分的开发者都不知道这些，因为一般情况下，他们只需要通过mutex这一类高级的同步锁，来在多线程下解决数据竞争问题。至于这些锁是如何实现的，对他们来说并不重要。但是如果要接触到更底层的原理，或需要编写`lock-free`数据结构，亦或者是需要做更高性能的优化时，那么我们就需要理解这些更接近「机器语言」的数据结构和设计。有时候，一些原子类型或者原子操作，甚至可以让代码缩减到只有1-2个CPU指令。

<!-- more -->

## 为什么会有这篇文章

最近在阅读`objc-block`代码，其中C++实现的 objc-block-trampolines.mm 部分，有一个`lock-free`的设计。

> 首先什么是`lock-free`？

字面意思，`lock-free`就是无锁，简单的解释就是：一个系统设计，做到**没有阻塞，没有锁**的情况下，实现多线程之间的数据同步。

> 然后接看下这段代码

下面剔除了无关的代码之后的核心源码部分。

``` cpp
// fixme C++ compilers don't implemement memory_order_consume efficiently.
// Use memory_order_relaxed and cross our fingers.
#define MEMORY_ORDER_CONSUME std::memory_order_relaxed

struct TrampolinePointers {
    std::atomic<TrampolinePointers *> trampolines{nil};

    TrampolinePointers *get() {
        return trampolines.load(MEMORY_ORDER_CONSUME);
    }

public:
    void Initialize() {
        if (get()) return;

        // This code may be called concurrently.
        // In the worst case we perform extra dyld operations.
        void *dylib = dlopen("./libobjc-trampolines.dylib",
                             RTLD_NOW | RTLD_LOCAL | RTLD_FIRST);
        if (!dylib) {
            _objc_fatal("couldn't dlopen libobjc-trampolines.dylib: %s",
                        dlerror());
        }

        auto t = new TrampolinePointers(dylib);
        TrampolinePointers *old = nil;
        if (! trampolines.compare_exchange_strong(old, t, memory_order_release))
        {
            delete t;  // Lost an initialization race.
        }
    }
};
```

这段代码摘自`objc`源码中的block实现部分，具体代码在 objc-block-trampolines.mm 内。

代码实现了一个无锁的单例，单例对象为`trampolines`。其中`get()`和`Initialize()`函数是可以多线程并发执行的。

`get()`无需解释，直接返回当前的单例对象，读操作，在多线程下本身也没什么问题。（PS：这种设计下单例，是**有可能**为空的）

`Initialize()`是一个双判初始化操作，其中关键代码是`trampolines.compare_exchange_strong(old, t, memory_order_release)`：

它将`trampolines`变量和`old`变量相比较，如果相等，那么就让`t`「替换」掉`trampolines`。最核心的一点，它是一个**原子操作**。虽然`Initialize()`函数可以有多线程并发执行，但是`compare_exchange_strong()`函数同时只有一个线程可以执行。

假设当前有两个线程在同时执行，在第一个线程将`trampolines`通过`compare_exchange_strong()`赋值后，第二个线程`compare_exchange_strong()`判断一定会返回false，所以第二个线程需要释放预先创建的`t`。

代码似乎可以解释，但是其中的`memory_order_release`和`MEMORY_ORDER_CONSUME(memory_order_relaxed)`是用来做什么的？

## 前序
### 谈谈指令乱序

大学的时候从课本上我们就已经学到，现代CPU早已经进入了多核时代，现代的编程技术，多线程也不再是一件稀奇的事情。分支预测，乱序执行，指令优化，都会让最终运行在机器上的代码，不再是我们编写代码的时候所预期的那样。

除了这些以外，各类编译器还会根据自己的实现和规则，对编译的目标代码进行优化，指令重排就是这些优化中重要的一项。而指令重排对代码的开发者来说完全黑盒。

下面用一段简单的代码来举个例子:
``` cpp
int A = 0;
int B = 0;

void func(void) {
    A = 1;
    if (B == 0) {
        A = 2;
    }
}
```

Clang编译上面这段代码后会产出如下的汇编：
![memory_reorder_case1](memory_reorder_case1.jpg)


我们把我们看到的理解代码的顺序，暂且叫`代码顺序`，而这种不按照`代码顺序`来执行的编译器行为称之为`指令乱序`。从编译器的角度来看，它只需要保证**在单线程的情况下，用户的代码(也就是我们写的代码)能够正确地执行，并且符合单线程的预期即可**。就上面的代码而言，「看起来」和「执行结果」的确是没有问题的。但是在多线程环境下，这个结论就不再成立了。因为**编译器是不知道多线程环境下代码执行的先后顺序，也不知道哪些变量会被共享**，`指令乱序`会导致严重的后果，比如下面这个反面例子

「锁」是绝大部分开发者在编程的时候，会选择用来解决多线程下的数据竞争问题。不论是mutex，还是自选锁，用锁来保护临界区，是一个程序员的必备本领。那么如果锁住的临界区内的代码，在被编译优化（指令乱序）后，理论上不也是会出现竞争问题？比如把上面的代码稍微改一下。

``` cpp
```

但是经验告诉我们，锁是肯定不会有问题的，因为线上的代码跑得好好的，回头再来解释这个问题。

### 内存可见性

我们知道，CPU执行指令的速度，和内存的读写速度不是一个量级的，因此CPU引入了多级缓存，就像下面这样（原谅我的画功）：

![2022-07-20_C++_memory_order](CPU_memory_struct.jpg)

每个内核都有自己的L1甚至L2缓存，为了提升运行速度，内核读某个内存值的时候，如果在缓存内命中，那么很有可能就会直接使用缓存内的值；同样的，写入操作也有可能只会临时写入缓存；当触发缺页，或者回写机制的时候，这些缓存中的值才会真正刷新到主存中去。这也就意味着，其实每一个内核都有自己独立的缓存，而各个缓存之间的同步机制，会影响内存的「可见性」。

举个例子 --- 图

上面这个例子中，当一个内核(线程)在读取变量之后，另一个内核(线程)对它做了修改，但没有回写主存，也没有同步到其他内核(线程)中去的时候，就会出现不一致的情况。这块内存对于不同的线程的可见性也就不一致了。

### 原子操作



## 内存排序

### 三种内存模型


## 解释上面的问题

> 为什么临界区的代码不会被重排序到临界区之外？

> 回头再来分析最开始的`lock-free`的代码

## 总结

看了很多网文，都把内存排序和内存一致性混淆在一起，这也是我一直没有理清楚的原因。直到我看了这几个回答，才确定一些概念：

> 关于内存可见性
**原子操作**除了是原子的，不可分割的以外，还有一点，它可以确保所有线程的可见性。只要确定A原子操作「Happens Before」B原子操作，那么B原子操作一定是可以读到A操作的值的。
> 关于内存排序



https://www.jb51.net/article/246087.htm
https://www.huliujia.com/blog/df3c2e8a9ef77bd2ed0d83292778734eb395970c/
https://www.huliujia.com/blog/f85f72a3b3e3018ffe9c9d3c15dda0f5db079859/
https://www.ccppcoding.com/archives/221
https://paul.pub/cpp-memory-model/

https://colobu.com/2014/12/19/an-introduction-to-lock-free-programming/

shared_ptr:
https://stackoverflow.com/questions/14881056/confusion-about-implementation-error-within-shared-ptr-destructor
https://stackoverflow.com/questions/49112732/memory-order-in-shared-pointer-destructor

### 内存模型

通常说的内存模型，包含两方面，一方面是「数据结构」，另一方面是「并发」。数据结构是最基础的，我们所说的对象，指针地址等等，都离不开数据结构。

### 内存序列模型

**对原子操作**，C++有6种内存序列，分别是：
1. `memory_order_relaxed`
2. `memory_order_consume`
3. `memory_order_acquire`
4. `memory_order_release`
5. `memory_order_acq_rel`
6. `memory_order_seq_cst`

在不指定的情况下，默认是`memory_order_seq_cst`。

虽然有6种内存序列，但他们只对应了3种模型，分别是：
1. `sequentially consistent`（序列一致性排序） 「`memory_order_seq_cst`」
2. `acquire-release`（获取-释放排序）「`memory_order_consume`, `memory_order_acquire`, `memory_order_release`, `memory_order_acq_rel`」
3. `relaxed`（松散排序）「`memory_order_relaxed`」


#### 顺序一致性的模型
上面提到了，内存排序默认就是`memory_order_seq_cst`，对应的模型就是「顺序一致性排序模型」。

顺序一致性，意思是在程序执行过程中，所有对于原子类型数据的操作，就像是我们从源码看上去，以我们所看到的那样。

换句话说，顺序一致性，在多线程的环境下，原子类型的数据，将会像在单线程下处理一样 —— 一个操作接一个操作排好顺序来执行。到目前为止，这个顺序是最好理解的，在多线程环境下，每一个线程，对原子数据进行操作的优先级都是一样的。

但是这也就意味着，多线程下对于这个原子的操作是无法被排序的。假如，线程A对原子的操作，必须在线程B对原子操作之后，那么这个排序就必须要让这两个线程都”知道“。

#### 非顺序一致性的模型

当进一步深入研究的时候，就会发现在「非顺序一致性」的模型下，多线程将会变得非常复杂。因为在这种模型下，就不再和上面一样有一个「全局有序的事件队列」了。在理解这种模型的时候，我们必须要抛开我们固有的多线程操作的心理预期模型，原子操作在不同线程中看到的将会是不同的样子。

##### 松散序列
松散序列是非顺序一致性中的一种，简单的来说，因为CPU的多级缓存存在，使用「`memory_order_relaxed`」这一种模型，会让不同的线程看到原子数据修改的不同阶段的值。

举个例子：
如果一个原子数据，在被一顿操作后，数据顺序为 `10, 8, 6, 4, 2`。从心理预期来看，`2`是最终的值，任何人，任何线程在这个原子操作被改为`2`之后，读取到的就应该是`2`。

实则不然，因为多级缓存的存在，在每一次使用「`memory_order_relaxed`」模式去修改值的时候，不会同步到所有的缓存。那就会导致，每一个线程去尝试读取的时候，当前这个线程所在的CPU核对应的若干级缓存还没有来得及更新，从而读到了一个历史的值，比如是`8`。当这个线程再去读的时候，读取到的**不可能**是旧值，显而易见，缓存肯定是往更新的值同步。

这里可以看到「`memory_order_relaxed`」这个模式，对原子数据的预期完全不可控，如非必要，完全不建议使用这个模式。