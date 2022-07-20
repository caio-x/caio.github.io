---
title: oc中@synchronized的性能瓶颈
date: 2019-12-20
tags: [object-c,锁]
categories: 语法糖
---

## @synchronized是什么？

一般来说OC上的语法糖，都可以使用 `clang --rewrite-objc code.m` 让clang 编译为 c 或 c++ ，由此可以窥探这些语法糖的具体实现(但是实际上XCode在编译的时候，并不会有rewrite的这个过程)。例如，[oc的block](https://caio.ink/oc-block-imp/)，@autoreleasepool，@synchronized等等

<!-- more -->

写一个简单的例子:
```c
int main(int argc, const char * argv[]) {
    NSObject *o = [NSObject new];
    @synchronized (o) {
        NSLog(@"helloworld");
    }
}
```

clang 重写后:
```c
int main(int argc, const char * argv[]) {
    NSObject *o = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
    {
        id _rethrow = 0;
        id _sync_obj = (id)o;
		
        objc_sync_enter(_sync_obj);
		
        try {
            struct _SYNC_EXIT {
                _SYNC_EXIT(id arg) : sync_exit(arg) {}
                ~_SYNC_EXIT()
                {
                    objc_sync_exit(sync_exit);
                }
                id sync_exit;
            } _sync_exit(_sync_obj);

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_vt_qgsk8yps0js52hgrjtwg9ch80000gn_T_main2_73ab4e_mi_0);
        } catch (id e) {
            _rethrow = e;
        }
		
        {
            struct _FIN {
                _FIN(id reth) : rethrow(reth) {}
                ~_FIN() {
                    if (rethrow)
                        objc_exception_throw(rethrow);
                }
                id rethrow;
            } _fin_force_rethow(_rethrow);
        }
    }
}
```

代码比较清晰：
1. `@synchronized` 整个代码块都被try包裹。
2. 执行try的之前调用 `objc_sync_enter(_sync_obj)` 进行加锁。
3. try代码块内先构建一个C++对象，C++对象的构造器内会传入"被加锁"的对象，并且被这个C++对象持有。
4. try代码块结束的时候，也就是C++对象析构的时候，会调用`objc_sync_exit(sync_exit)`，解锁这个"被加锁"的对象。
5. 最终在`@synchronized` 代码块结束的时候，如果catch了异常就再原样把异常抛出去。

> 上面描述的时候，对`被加锁`三个字加了引号。OC对象可以被加锁么？先暂且这么说，后文再对此进行解释。

那么现在问题就转变为：
1. objc_sync_enter(obj); 加锁
2. objc_sync_exit(obj); 解锁

### objc_sync_enter 和 objc_sync_exit

在`libobjc.A.dylib`(就是俗称的Runtime)中，有一个 objc-sync.h/.m 文件 实现了这两个函数。

```c
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
    }

    return result;
}

int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	
    return result;
}
```

这个代码也很容易理解，通过`id2data`,取到与 obj 对应的 `SyncData`,然后对这个数据结构中的 mutex 进行加解锁操作。也就是说`@synchronized`加锁的对象是与oc对象绑定的一个mutex锁？

## id2data

在描述具体实现之前，需要了解以下几个结构体：
#### SyncData
```c
typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData;
    DisguisedPtr<objc_object> object;
    int32_t threadCount;  // number of THREADS using this block
    recursive_mutex_t mutex;
} SyncData;
```
1. SyncData是用来描述每一个"被加锁"的对象。
2. SyncData是一个链表，由nextData指向下一个节点。
3. object指向这个"被加锁"的对象。
4. threadCount记录了正在使用这个节点对象的线程（当没有线程使用这个节点的时候，它才会被释放）。这里的使用是指已经加锁，或者正在等待锁。
5. mutex 递归锁(os_unfair_recursive_lock),真正加锁、解锁的锁对象。

#### SyncCache
```c
typedef struct SyncCache {
    unsigned int allocated;
    unsigned int used;
    SyncCacheItem list[0];
} SyncCache;
```
1. SyncCache 是存储在TLS(本地线程存储)中的，用来记录当前线程所有已经持有锁的`SyncData`。（排除TLS中快速缓存的那一个，下文会具体说）
2. list是一个`SyncCacheItem`数组,`SyncCacheItem`里就只有两个东西，一个是`SyncData`，另一个就是lockCount，lockCount记录了当前线程对这个对象加锁次数。（@synchronized是一个可重入的递归锁）
3. allocated是已分配的cache大小，used是已使用大小。默认情况下，每个线程首次创建的时候会有4个size的容量。当每次插槽不够的时候会指数级扩大这个大小。


id2data的代码比较长，简单来说这个方法的目的就是，获取到这个id对象所对应(绑定)的SyncData,然后...调用syncData.mutex.lock()

**那么这个SyncData在哪呢？应该怎么去取它呢？**

简单概括一下：
1. `TLS`:大部分情况下，开发者在某一个线程对某一个对象的同步锁中，再嵌套其他对象的同步锁的概率，要远低于只用一个同步锁。（**其实有多个锁嵌套的时候，这时候是不是应该要考虑代码结构有问题了**）所以苹果在设计这个锁的时候，会优先在`TLS`中预留一个`SyncData`。即优先向`TLS`中读取或者写入，注释上称它为fast cache。
2. `SyncCache`中:上面已经提及，对于每一个线程已经获得的锁，都会在`TLS`中存储的`SyncCache`结构体中。`SyncCache`中的数组存储的就是当前线程已经获取的锁对应的`SyncData`了。
3. 全局数据中： 如果在1，2中都没有找到，那么说明当前线程尚未对这个对象持有锁。`static StripedMap<SyncList> sDataLists`这个全局数据结构记录了进程内所有已经被持有同步锁或者曾经被持有过的对象。`sDataList`是个重写的Map，key是同步锁对象，value是`SyncList`。

说明一下这个 aDataLists：
- StripedMap通过对key(id对象)的地址做一了一系列偏移和对mapsize的求模操作，把对象散列在64个（iPhone上是8个）插槽中。每个插槽内是一个`SyncData`链表，即`SyncList`结构体。`SyncList`内有一个`spinlock_t`,也就是对这个链表进行操作的时候的锁。
- 另外如果在StripedMap没有找到这个对象，那么会从SyncList中找一个已经释放的`SyncData`(threadCount==0，即既没有线程持有，也没有线程在等待)来复用


最后总结一下`id2data`这个函数的操作步骤：
1. 通过 `SYNC_DATA_DIRECT_KEY`和`SYNC_COUNT_DIRECT_KEY`这两个key优先从TLS中查询`SyncData`,根据操作类型（加、解）分别修改TLS中的值，返回`SyncData`。
2. 通过遍历TLS中的`SyncCache`数组，找到该对象的`SyncData`，根据操作类型（加、解）分别修改`SyncCacheItem`中的lockCount，返回`SyncData`。
3. 1，2都没有，那么去找其他线程已经持有或者曾经被持有过同步锁的`SyncData`对象：从`sDataLists`中找到当前需要加同步锁对象对应散列的插槽`SyncList`，从`SyncList`中找到这个对象对应的或者复用的`SyncData`。
4. 如果第3步都没有找到（说明是整个进程内第一次加锁），那就创建一个`SyncData`。并且加到`SyncList`的队首。
5. 如果`TLS`中`SYNC_DATA_DIRECT_KEY`是空的，那么优先保存。**否则**保存到`SyncCache`中

注:
1，2两步骤中在lockCount减小到0的时候，也就是当前线程需要释放SyncData（实际并不会释放）的时候。会同时修改`SyncData`中的threadCount,这个操作必须是原子操作。因为前者操作必然是在当前线程，但是后者操作可能是多线程的。（以及3步骤中，在持有并返回其他线程已经加同步锁的`SyncData`对象。）

### @synchronized的性能到底差在哪？？

很多文章在对比各种锁的性能，都提到了@synchronized性能是最差的。那么到底差在哪呢？

上文已经具体分析了@synchronized的所有实现逻辑了，但是可能并没有一个概念，到底差在哪？

怀疑以下的两个点：
1. `id2data`即在`TLS`,`SyncCache`,`sDataLists`寻找对应同步锁对象的过程
2. `SyncData`中的锁。

先做一个实验对比一下同步锁有多慢。

**以下实验只是为了做一个定性的分析，所有数据只提供一个参考。**

```c
void main(){
		NSObject *t = [NSObject new];
		uint64_t pre = mach_absolute_time();
        int count = 0;
        for (int index = 0; index < 10000000; index++) {
            @synchronized (t) {
                count ++;
            }
        }
        uint64_t now = mach_absolute_time();
        uint64_t deltaA = now-pre;
        NSLog(@"synchronized count:%d time:%llu", count, deltaA/1000000);
	
        uint64_t pre1 = mach_absolute_time();
        count = 0;
        pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
        for (int index = 0; index < 10000000; index++) {
            pthread_mutex_lock(&mutex);
            count ++;
            pthread_mutex_unlock(&mutex);
        }
	
        uint64_t now1 = mach_absolute_time();
	
        uint64_t deltaB = now1-pre1;
        NSLog(@"count:%d time:%llu", count, deltaB/1000000);
}
```
分别对 pthread_mutex 和 synchronized 加锁 一百万次。
pthread_mutex:200ms
synchronized:6800ms

似乎...em...差了不是一点点呢...

因为上面的测试代码中，同步锁，每一次循环都会执行一遍`加锁..存入TLS..释放..从TLS移除`。我们试试在最外层套一个同步锁,这样每次都会在`TLS`中命中。

```c
void main(){
		NSObject *t = [NSObject new];
	@synchronized(t){
		uint64_t pre = mach_absolute_time();
        int count = 0;
        for (int index = 0; index < 10000000; index++) {
            @synchronized (t) {
                count ++;
            }
        }
        uint64_t now = mach_absolute_time();
        uint64_t deltaA = now-pre;
        NSLog(@"synchronized count:%d time:%llu", count, deltaA/1000000);
	
        uint64_t pre1 = mach_absolute_time();
        count = 0;
        pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
        for (int index = 0; index < 10000000; index++) {
            pthread_mutex_lock(&mutex);
            count ++;
            pthread_mutex_unlock(&mutex);
        }
	
        uint64_t now1 = mach_absolute_time();
	
        uint64_t deltaB = now1-pre1;
        NSLog(@"count:%d time:%llu", count, deltaB/1000000);
	}
}
```
pthread_mutex:200ms
synchronized:2700ms

似乎..快了..不少呢..

但是这样测试下去，没法获得准确的结果。我们直接调试Runtime的源码，通过修改源码来“控制变量”。

----
这里有一个小插曲:

在尝试通过修改源码的来实验的时候，`synchronized`的执行时间总是出奇的长。于是我把objc-sync.mm中 `objc_sync_enter` 和 `objc_sync_exit`方法改为空操作，直接返回。

结果，synchronized的执行时间居然还维持在1000ms+。

汇编代码显示，还有一个objc_retain方法。
![](https://blog-caio.oss-cn-hongkong.aliyuncs.com/blog/2019/12/1ff8b0b7aa6cc03e763e5e3c693c19bf.png)

也就是说，编译器在synchronized代码块中还会自动插入代码，持有这个同步锁对象。

因为测试代码不涉及到多线程间对象的持有和释放，所有我们直接把它改为mrc。

执行时间变为 40ms+，objc_retain也确实没有了。
![](https://blog-caio.oss-cn-hongkong.aliyuncs.com/blog/2019/12/6c4e55e070f1d49f795e3a440fc2cffd.png)

也就是说，引用计数占了1000ms+。以下实验均在mrc下

#### 同步锁对象寻找过程

修改源码，让 `objc_sync_enter` 和 `objc_sync_exit`方法只执行查询操作，不执行锁操作。

整个查询过程的执行时间在 900ms+

#### 锁过程

同步锁最终还是使用的`SyncData`中的`os_unfair_recursive_lock`实现的递归锁。

这个锁是苹果用来代替已废弃的 OSPinLock 的，OSPinLock让CPU空转，而不会让出时间片，在GCD的线程优先级上会出现饥饿或者死锁问题。具体的可以看[这篇文章](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)。

`os_unfair_lock`锁的线程会处于休眠状态，从用户态切换到内核态，而并非忙等
![](https://caio.ink/wp-content/uploads/2019/12/e9b18665440cf264a15213547cfc6860.png)

直接测试一下，递归锁的加锁解锁效率。
```c
typedef struct os_unfair_recursive_lock_s {
    os_unfair_lock ourl_lock;
    uint32_t ourl_count;
} os_unfair_recursive_lock;
```
``` c
        os_unfair_recursive_lock lock = ((os_unfair_recursive_lock){OS_UNFAIR_LOCK_INIT, 0});
        os_unfair_recursive_lock *mlock = &lock;
        for (int index = 0; index < 10000000; index++) {
            os_unfair_recursive_lock_lock_with_options(&lock, 0x00000000);
                count ++;
            os_unfair_recursive_lock_tryunlock4objc(&lock);
        }
```

如果只是锁的话，执行时间只有200ms。`os_unfair_lock`的执行效率比`pthread_mutex`略高。而且当执行次数越多，这个差距稳定在10%

引用计数 1000ms+同步锁对象查询900ms+递归锁200ms = 2100ms

距离第二次的实验还差500ms左右。500ms花费在哪？

objc-os.h有一个LOCKDEBUG的宏，在debug期间会有一些耗时判断，把宏关闭之后，上面的第二次实验的时间就从2900ms -> 2100ms左右，与上面的计算就对应上了。

> 总结一下:
1.synchronized同步锁使用的 os_unfair_lock 本身性能是不差的，甚至优于pthread_mutex。
2.synchronized的性能，主要是花费在实现这个语法糖上。引用计数的管理和查找与对象绑定的同步锁消耗了绝大部分的时间。而且上面的实验我们还只是在TLS内操作了，如果一个线程内持有的同步锁个数越多，那么将花费在Cache上和全局数据内的查询时间也会更多。