---
title: "聊一聊Go的原生高性能对象复用池sync.Pool"
date: 2022-04-23
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/sync-pool-icon.png
categories:
- Go
- 缓存
tags:
- 对象池
- 伪共享
- sync.Pool

metaAlignment: center
---

池化操作在计算机领域层出不穷，如连接池、线程\协程池、工作池。我们要知道，之所以引入资源池，本质上是为了复用资源。
<!--more-->

![对象池](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/sync-pool-cover.png)

## 前言
池化操作在计算机领域层出不穷，如连接池、线程\协程池、工作池。我们要知道，之所以引入资源池，本质上是为了复用资源，减少多余的申请或者创建操作，事先分配好部分资源并把它管理起来。今天来聊一聊```Go```原生库```sync.Pool```的使用场景和内部实现。
## Sync.Pool解决了什么问题？
先说结论，它是一个临时存储，可复用的对象池，适用于频繁申请使用的对象，常常用来减轻GC负担。

## Sync.Pool的用法
### 常见用法：

```go
func TestSyncPool(t *testing.T) {
    // 定义对象初始化的方式
    mp := &sync.Pool{
        New: func() interface{} {
            t.Log("create instance")
            return 0
        }}

    // 获取
    ist := mp.Get()
    t.Log("ist = ", ist)
    ist = 10

    // 归还
    mp.Put(ist)
    ist = mp.Get()
    t.Log("ist = ", ist)

    mp.Put(ist)
    runtime.GC()
    // 执行gc之后，对象会暂存到victim，关于victim下文会解释这里先理解成一个回收站
    ist = mp.Get()
    t.Log("ist = ", ist)
}
```
**运行：**
```bash
=== RUN   TestSyncPool
    heypool_test.go:12: create instance
    heypool_test.go:17: ist =  0
    heypool_test.go:22: ist =  10
    heypool_test.go:27: ist =  10  (如果获取的P和Put()操作的P是同一个,则首次GC之后仍然可以拿到10)
--- PASS: TestSyncPool (0.00s)
PASS
```

看起来似乎和缓存池用法无异，但是我们看官方Get()函数的注释，

> Get may choose to ignore the pool and treat it as empty.  
 Callers should not assume any relation between values passed to Put and
 the values returned by Get.
> 

这里我们先不深入探究，仅仅看注释，告知我们不能在put和get两个操作作关联假设，由于**我们无法保证GC触发和对象池的回收状态，因此sync.Pool不适合作为数据库连接池等场景。**

### 官方示例：

这是一个官方测试用例的写法，我们可以预测```sync.Pool```的使用意图

```go
func TestPool(t *testing.T) {
    // 临时禁用系统触发GC
    defer debug.SetGCPercent(debug.SetGCPercent(-1))
    var p Pool
    if p.Get() != nil {
        t.Fatal("expected empty")
    }

    // 让运行协程挂在当前P上面，确保对象池和程序读写一一对应
    Runtime_procPin()
    p.Put("a")
    p.Put("b")
    if g := p.Get(); g != "a" {
        t.Fatalf("got %#v; want a", g)
    }
    if g := p.Get(); g != "b" {
        t.Fatalf("got %#v; want b", g)
    }
    if g := p.Get(); g != nil {
        t.Fatalf("got %#v; want nil", g)
    }
    Runtime_procUnpin()

    for i := 0; i < 100; i++ {
        p.Put("c")
    }
    // 由于存在victim过度结构，在gc之后对象池还没清空
    runtime.GC()
    if g := p.Get(); g != "c" {
        t.Fatalf("got %#v; want c after GC", g)
    }
    // 从测试代码断言来看，第二次gc会真正淘汰victim
    runtime.GC()
    if g := p.Get(); g != nil {
        t.Fatalf("got %#v; want nil after second GC", g)
    }
}
```
### 基准测试
我们来看下引入```sync.Pool```前后性能的对比：
```go
type Pixel struct {
    a int
}

var pool = sync.Pool{
    New: func() interface{} { return new(Pixel) },
}

//go:noinline
func inc(s *Pixel) { s.a++ }

func BenchmarkWithoutPool(b *testing.B) {
    var s *Pixel
    for i := 0; i < b.N; i++ {
        s = &Pixel{a: 1}
        b.StopTimer()
        inc(s)
        b.StartTimer()
    }
}

func BenchmarkWithPool(b *testing.B) {
    var s *Pixel
    for i := 0; i < b.N; i++ {
        s = pool.Get().(*Pixel)
        s.a = 1
        b.StopTimer()
        inc(s)
        b.StartTimer()
        pool.Put(s)
    }
}
```
**输出：**
```bash
$ go test -bench=.
goos: windows
goarch: amd64
pkg: HelloGo/basic/pool/sync
cpu: Intel(R) Core(TM) i7-4710MQ CPU @ 2.50GHz
BenchmarkWithoutPool-8           4817011               271.9 ns/op
BenchmarkWithPool-8             12893811               107.9 ns/op
PASS
ok      HelloGo/basic/pool/sync 472.933s
```

## Sync.Pool是如何设计的？
在探究内部实现之前，我们先来提取一些概念。
### 1. GMP协作

我们都知道在Go中协程的调度的GMP模型（可参考专栏之前的总结笔记：[聊一聊操作系统线程调度与Go协程](https://juejin.cn/post/6844904052686323725)），每个处理核心**P**都有自己的协程**G**任务队列，```sync.Pool```底层的```Get()```对象获取函数，就是通过本地私有队列和共享队列进行获取的。

```go
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    // 指定当前访问的P，确保在同一个对象池
    l, pid := p.pin()
    x := l.private
    l.private = nil
    if x == nil {
        // 如果当前私有队列为空，则尝试从共享池pop出
        x, _ = l.shared.popHead()
        if x == nil {
            // 共享池为空，则去其他P池窃取
            x = p.getSlow(pid)
        }
    }
    runtime_procUnpin()
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    }
    // 私有空间获取为空，窃取为空，则调用初始化构造对象
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

### 2. 多级缓存

在操作系统组件中，从CPU寄存器到物理磁盘，从一个CPU周期到L1/2/3缓存，再到主存、再到文件系统，单次访问的操作时间量级愈发增大，多级缓存这个思路在计算机领域很早就被引入。

> As hardware architecture and technology advanced, processor performance and frequency grew much faster than memory cycle times, leading to a large gap in performance. The problem of increasing memory latency, relative to processor speed, has been dealt with by adding high speed cache memory.
> 
> 
> > 随着硬件升级CPU处理速度已经远超缓存加载周期，所以尽量让CPU快速取到缓存能有效提升性能。
> > 

我们来看下```sync.Pool```的内部成员：

```go
type Pool struct {
    noCopy noCopy

    local     unsafe.Pointer // 当前P指定的对象地址
    localSize uintptr        // size of the local array

    victim     unsafe.Pointer // 当前P首次淘汰的过度区间
    victimSize uintptr        // size of victims array
    
    New func() interface{}
}
```

通过```victim```牺牲池，过度暂存首次被回收的对象，让程序在申请对象之前尽量复用，使用起来更加“**平滑**”。

```go
func init() {
    // sync.Pool启动注册GC执行逻辑
    runtime_registerPoolCleanup(poolCleanup)
}

func poolCleanup() {
    // 当GC触发执行时候，把回收站的旧对象池清空
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }

    // 把本来要被淘汰的对象暂存到victim
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }

    oldPools, allPools = allPools, nil
}
```

在启动时候注册GC处理逻辑，可以看到```victim```机制其实借鉴了其他语言的**分代回收**思想，在清理对象之前会暂存起来，提高复用的几率。

---

经过上面的拆解，我们可以得知在多个P的对象池获取优先级，如下图所示：

![sync-pool-get-seq.png](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/sync-pool-get-seq.png)

可以看到由于每个P都是独立的，所以①私有池、②牺牲池、⑤初始化创建是当前**P**独占的，只有③是有可能被其他P所共享的，所以关于③共享的操作需要加锁。

### 3.无锁Lock-free

我们来看下```Sync.Pool```是怎么处理③的

```go
func (d *poolDequeue) popHead() (interface{}, bool) {
    var slot *eface
    for {
        // 获取原子变量判断当前共享队列的区间
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if tail == head {
            return nil, false
        }       
        head--
        ptrs2 := d.pack(head, tail)
        if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
            // 使用CAS，基于乐观期望的自旋锁设计           
            slot = &d.vals[head&uint32(len(d.vals)-1)]
            // 获得执行权后跳出
            break
        }
    }
    // ...
    return val, true
}
```
通过使用乐观期望的自旋锁，比读写锁更加的轻量级，达到近似无锁的设计。

### 4. **伪共享**

第四个概念是关于伪共享，什么是伪共享呢，其实说的是当我们要更新一个变量的值时候，CPU在读取加载内存块时(行加载)，由于局部性访问原理，在寻址空间是连续的情况下，加载变量进行操作时会同时覆盖到其他变量地址空间，这个时候每个P要顾虑到我的变量有没有被其他变量的连续区间影响，**因为同一块地址空间中，当有并发读写时候会产生冲突，如下图所示：**

![false-share.png](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/false-share.png)

为了避免，可以想到的有两种处理方式：

- 排他互斥：即每次保证只有一个**P**进行操作，独占式处理，对性能有较大影响
- 内存对齐：即使用填充位，在存储变量的时候，给连续地址空间填上占位符，保证当前操作的变量区间和指令寻址空间是一致的，即不用担心其他变量地址越界问题。
    
    ![fixed-false-share.png](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/fixed-false-share.png)
    

个人觉得第二种方式更加符合```Go```并发的哲学处理：**不要用共享内存来通信，而要用通信来共享内存。**

那在```sync.Pool```是怎么做的呢？

```go
// Local per-P Pool appendix.
type poolLocalInternal struct {
    private interface{} // 私有P标志位
    shared  poolChain   // 本地P指向的队列
}

type poolLocal struct {
    poolLocalInternal

    // 填充字节数组，使用128倍数来访问地址区间
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

可以看到```poolLocal```有个```pad```内部成员，其中```poolLocal``` 其实是```sync.Pool```内部local指针指向的内存块，通过128大小区间(64位系统的倍数)来访问，每次对象池寻址都是和CPU单次指令操作区间是一致的，这样子从而避免了“伪共享问题”，如上图所示。

## 总结：

以上我们从GMP模型、多级缓存、伪共享等概念对```Sync.Pool```进行分析，知道了其内部的获取优先级，也知道了它是如何解决多个**P**冲突，并使用**CAS**来减轻锁操作的负担的。

可以感觉到**无论是从宏观上还是细节实现，Go都是在默默地遵循着一定的设计规范。**

## 参考链接
- Explore Go sync.Pool as Cache  
https://medium.com/geekculture/go-sync-pool-as-cache-in-kubernetes-4e247c52e732
- 深度分析 Golang sync.Pool 底层原理  
https://www.cyhone.com/articles/think-in-sync-pool/
- Go: Understand the Design of Sync.Pool  
https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277