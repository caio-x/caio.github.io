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

它将`trampolines`变量和`old`变量相比较，如果相等，那么就让`t`「替换」掉`trampolines`，其中最核心的一点，它是一个**原子操作**。虽然`Initialize()`函数可以有多线程并发执行，但是`compare_exchange_strong()`函数同时只有一个CPU可以执行，并且这个操作是不能被中断的。

假设当前有两个线程在同时执行，在第一个线程将`trampolines`通过`compare_exchange_strong()`赋值后，第二个线程`compare_exchange_strong()`判断一定会返回false，所以第二个线程需要**释放**已创建的`t`。

代码似乎可以解释，但是其中的`memory_order_release`和`MEMORY_ORDER_CONSUME(memory_order_relaxed)`是用来做什么的？

## 前序
### 谈谈指令乱序

大学的时候从课本上我们就已经学到，现代CPU早已经进入了多核时代，现代的编程技术，多线程也不再是一件稀奇的事情。分支预测，乱序执行，指令优化，都会让最终运行在机器上的代码，不再是我们编写代码的时候所预期的那样。

除了这些以外，各类编译器还会根据自己的实现和规则，对编译的目标代码进行优化，指令重排就是这些优化中重要的一项，但指令重排对这些代码的开发者来说完全黑盒。

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

我们把我们看到的理解代码的顺序，暂且叫`代码顺序`，而这种不按照`代码顺序`来执行的编译器行为称之为`指令乱序`。从编译器的角度来看，它只需要保证**在单线程的情况下，代码能够正确地执行，并且符合单线程的预期即可**。就上面的代码而言，「看起来」和「执行结果」的确是没有问题的。但是在多线程环境下，这个结论就不再成立了。因为**编译器是不知道多线程环境下代码执行的先后顺序，也不知道哪些变量会被共享**，`指令乱序`会导致严重的后果。看下面这个反面例子

「锁」是绝大部分开发者在编程的时候，会选择用来解决多线程下的数据竞争问题。不论是mutex，还是自选锁，用锁来保护临界区，是一个程序员的必备本领。那么按照上面编译器不可预知的`指令乱序`行为，如果锁住的临界区内的代码，在被编译优化后，理论上不也是会出现竞争问题？比如我们把上面的代码稍微改一下。

``` cpp
```

但是经验和线上的代码告诉我们，锁，是肯定不会有问题的，至于为什么，回头再来解释这个问题。

### 内存可见性

我们知道，CPU执行指令的速度，和内存的读写速度不是一个量级的，因此CPU引入了多级缓存，就像下面这样（原谅我的画功）：

![2022-07-20_C++_memory_order](CPU_memory_struct.jpg)

每个内核都有自己的L1，L2甚至L3缓存，为了提升运行速度，在内核读某个内存值的时候，如果在缓存内命中，那么很有可能就会直接使用缓存内的值；同样的，写入操作也有可能只会临时写入缓存；当触发缺页，或者回写机制的时候，这些缓存中的值才会真正刷新到主存中去。垂直来看，其实每一个内核都有自己独立的缓存，而各个缓存之间的同步机制，会影响内存的「可见性」。

举个例子 --- 图

上面这个例子中，当一个内核(线程)在读取变量之后，另一个内核(线程)对它做了修改，但没有回写主存，也没有同步到其他内核(线程)中去的时候，就会出现不一致的情况。这块内存对于不同的线程的可见性也就不一致了。

### 原子操作

原子操作，就是在单核CPU上，一个CPU指令就能完成的操作，比如MOV，INC等等。如果一个操作需要多个指令来完成，那么在多个指令之间，有可能会触发系统中断，在中断过程中，修改了这个操作的中间变量，这个操作就不再「原子」。


## 内存排序

### 三种内存模型

**对原子操作**，C++有6种内存序列，分别是：
1. `memory_order_relaxed`
2. `memory_order_consume`
3. `memory_order_acquire`
4. `memory_order_release`
5. `memory_order_acq_rel`
6. `memory_order_seq_cst`

在不指定的情况下，默认是`memory_order_seq_cst`。

虽然有6种内存序列，但他们只对应了3种模型，分别是：
1. `sequentially consistent`（顺序一致性排序） 「`memory_order_seq_cst`」
2. `acquire-release`（获取-释放序列）「`memory_order_consume`, `memory_order_acquire`, `memory_order_release`, `memory_order_acq_rel`」
3. `relaxed`（松散序列）「`memory_order_relaxed`」


#### 顺序一致性的模型
上面提到了，内存排序默认就是`memory_order_seq_cst`，对应的模型就是「顺序一致性排序模型」。

顺序一致性，意思是在程序执行过程中，所有对于原子类型数据的操作，就像是我们从源码看上去，以我们所看到的那样。

换句话说，顺序一致性，在多线程的环境下，原子类型的数据，将会像在单线程下处理一样 —— 一个操作接一个操作排好顺序来执行。到目前为止，这个顺序是最好理解的，在多线程环境下，每一个线程，对原子数据进行操作的优先级都是一样的。

但是这也就意味着，多线程下对于这个原子的操作是无法被排序的，也就是说CPU是无法针对这些原子操作的前后进行编译优化的。

#### 非顺序一致性的模型
非顺序一致性的模型，包含两种，一种是「松散序列」，另一种是「获取-释放序列」。

在这种模型下，就不再和上面一样有一个「全局有序的事件队列」了，细想一下这个模型在多线程下就会变得非常复杂了。在理解这种模型的时候，我们必须要抛开我们固有的多线程操作的心理预期模型，原子操作在不同线程中看到的将会是不同的样子。

##### 松散序列

松散序列是非顺序一致性模型中的一种，简单的来说，因为`指令乱序`的存在，使用「`memory_order_relaxed`」这一种模型，会让这个原子操作，「可能」出现在这个原子操作序列的任一阶段。

举一个例子来理解

```cpp
std::automic<int> x = 0;
std::automic<int> y = 0;

// 线程1
r1 = y.load(std::memory_order_relaxed); // A
x.store(r1, std::memory_order_relaxed); // B

// 线程2
r2 = x.load(std::memory_order_relaxed); // C 
y.store(42, std::memory_order_relaxed); // D
```

线程1 的操作为 A -> B, 线程B的操作为 C -> D, 所以如果按照顺序一致性的模型来理解的话，那么就会有下面的几种可能性
1. A -> B -> C -> D
2. A -> C -> B -> D
3. A -> C -> D -> B
4. C -> D -> A -> B
5. C -> A -> D -> B
6. C -> A -> B -> D

这6种可能性执行完之后，r1=r2=0, 或 r1 = 42,r2 = 0;

但是在松散序列下就不一样了，线程2中，y原子操作并不依赖任何操作，所以编译器可以选择将 D 操作顺序移动到 C 操作之前。
所以，在上面的6种可能性中，C，D可以交换，产生12种可能性。结果中可能会产生 r1=42,r2=42 这种情况。

再举个例子，这个例子来自「C++ Concurrency In Action」

```cpp
#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x_then_y()
{
    x.store(true,std::memory_order_relaxed); // A
    y.store(true,std::memory_order_relaxed); // B
}

void read_y_then_x()
{
    while(!y.load(std::memory_order_relaxed)); // C
    if(x.load(std::memory_order_relaxed)) { // D
        ++z;
    }
}

int main() {
    x=false;
    y=false;
    z=0;
    std::thread a(write_x_then_y);
    std::thread b(read_y_then_x);
    a.join();
    b.join();
    assert(z.load()!=0);
}
```

x, y在同一个线程内「依次」原子写入 true。另一个线程内，在y写入true之后，判断x是否已经是true了，如果是则z++。

从顺序一致性的模型来理解的话，断言是肯定不会触发的，因为在y变为true的时候，x一定已经是true了。

但是同样的，在松散序列下，x,y 没有任何相互依赖关系。A，B操作是有可能会指令乱序的。所以在这个case下，断言是有可能触发的。


到这里可以看到「`memory_order_relaxed`」这个模式，对原子数据的预期完全不可控，如非必要，完全不建议使用这个模式。

##### 获取-释放序列
上面两个序列是两个极端，一种是非常严格的策略，另一种则是非常宽松，而`获取-释放序列`则是位于两者之间。

## 解释上面的问题

> 为什么锁内临界区的代码不会被重排序到临界区之外？



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


