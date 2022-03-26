---
title: "聊一聊Go的协程泄露"
date: 2021-09-11
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/routine.PNG
categories:
- Go
- 后端
tags:
- 协程
- debug

metaAlignment: center
---

协程在Go里面是一个常见的概念，伴随着go程序的生命周期开始至结束。今天来聊一聊Go的协程泄露。  
<!--more-->

![协程通信](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/road.PNG)

## 前言
协程在Go里面是一个常见的概念，伴随着go程序的生命周期开始至结束。今天来聊一聊Go的协程泄露。  
关于协程的底层概念，GMP模型等，之前在前面博客已经作过分析，感兴趣可以看下之前的文章[聊一聊操作系统线程调度与Go协程](https://pixelpig.tech/2020/10/%E8%81%8A%E4%B8%80%E8%81%8A%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6%E4%B8%8Ego%E5%8D%8F%E7%A8%8B/)

## 泄露案例
关于协程泄露很常见的原因在于，有时候我们常常会忽略问题，直到资源负载异常才引起总是。
之前排除生产环境异常的时候，曾经遇到过go程序内存泄露的场景，内存泄漏和协程泄露有很大关系，本质上都是资源不回收导致的。这里简单还原一下现场，细节已经经过删减，大概形式如下：

```go
func JumpForSignal() int {
    ch := make(chan int)
    go func() {
        ch <- bizMtx
    }()

    go func() {
        ch <- bizMtx
    }()

    go func() {
        ch <- bizMtx
    }()
    //一有输入立刻返回
    return <-ch
}

func main() {
    // ...
    JumpForSignal()
    // ...
}
```

事后分析这个demo可以得知，这个函数调用会阻塞两个子协程，预期只有一个协程会正常退出。

## 获取协程信息
既然存在协程泄露，我们在日常工作怎么避免或者发现它呢？下面我们列举几个思路。
### 遵守准则
由于Go是自带GC的语言，很多时候写代码不需要关心变量的资源释放，不像C程序员变量申请之后需要在结束处释放。但是Go的```chan```在使用时候是有一些准则的，当确定chan不再使用时候，可以在输出方进行close，避免其他协程还在等待该```chan```的输出。
### 协程数量
找到泄露的协程，第一个能够想到的是协程数量，当你的函数处理逻辑比较简单，除了主协程之外，预期协程应该都在结束前返回，可以在main函数结束处调用```runtime```包的函数：

```go
// NumGoroutine returns the number of goroutines that currently exist.
func NumGoroutine() int {
    return int(gcount())
}
```
通过它可以返回当前协程总数量：
```go
func Count()  {
    fmt.Printf("Number of goroutines:%d\n", runtime.NumGoroutine())
}

func main() {
    defer Count()
    Count()
    JumpForSignal()
}
```
输出：
```bash
Number of goroutines:1
Number of goroutines:3
```

### 协程函数栈
还有一种比较常见定位协程的形式，在Go里面，可以用于分析协程函数的上下文，常见的比如go自带的pprof也是通过这种方式获取。  

下面来看一个示例，我们在上面的例子加一个```http```端口监听，用于接入go自带的pprof分析工具。

随后在浏览器输入：
```go
http://localhost:8899/debug/pprof/goroutine?debug=1
```
可以得到整个程序的协程列表：
```bash
goroutine profile: total 7
1 @ 0x165eb6 0x126465 0x126235 0x29341e 0x19de01
#   0x29341d    pixelgo/leak.JumpForSignal.func1+0x3d   F:/code/pixelGo/src/pix-demo/leak/leak.go:24

1 @ 0x165eb6 0x126465 0x126235 0x29347e 0x19de01
#   0x29347d    pixelgo/leak.JumpForSignal.func2+0x3d   F:/code/pixelGo/src/pix-demo/leak/leak.go:28

1 @ 0x165eb6 0x15bb3d 0x1975a5 0x228d05 0x229d8d 0x22c40d 0x321765 0x33437c 0x447c89 0x285239 0x285606 0x4493f3 0x450da8 0x19de01
#   0x1975a4    internal/poll.runtime_pollWait+0x64 D:/dev/go1.16/src/runtime/netpoll.go:227
#   0x228d04    internal/poll.(*pollDesc).wait+0xa4 D:/dev/go1.16/src/internal/poll/fd_poll_runtime.go:87
#   0x229d8c    internal/poll.execIO+0x2ac      D:/dev/go1.16/src/internal/poll/fd_windows.go:175
#   0x22c40c    internal/poll.(*FD).Read+0x56c      

// ...
```
结论是：当前程序一共有7个协程，可以看出分别有1个协程分配在```F:/code/pixelGo/src/pix-demo/leak/leak.go:24``` 和```F:/code/pixelGo/src/pix-demo/leak/leak.go:28```，正是上文泄露的代码块。

有时候还可以多维度去分析，比如输入：
```go
http://localhost:8899/debug/pprof/goroutine?debug=2
```
可以通过协程后面的标签,看到当前协程的不同状态，**running/io wait/chan send**
```go
goroutine 9 [running]:
runtime/pprof.writeGoroutineStacks(0x7f7d00, 0xc0000aa000, 0x0, 0x0)
    D:/dev/go1.16/src/runtime/pprof/pprof.go:693 +0xc5
net/http/pprof.handler.ServeHTTP(0xc000094011, 0x9, 0x7fba40, 0xc0000aa000, 0xc000092000)
    //..

goroutine 1 [IO wait]:
internal/poll.runtime_pollWait(0x223debb10d8, 0x72, 0xc000152f48)
    D:/dev/go1.16/src/runtime/netpoll.go:227 +0x65
internal/poll.(*pollDesc).wait(0xc0001530b8, 0x72, 0x93b400, 0x0, 0x0)
    //...

goroutine 6 [chan send]:
pixelgo/rout.JumpForSignal.func1(0xc000053800)
    F:/code/pixelGo/src/pix-demo/rout/leak.go:25 +0x10e
created by pixelgo/rout.JumpForSignal
    F:/code/pixelGo/src/pix-demo/rout/leak.go:23 +0x71

goroutine 7 [chan send]:
pixelgo/rout.JumpForSignal.func2(0xc000053800)
    F:/code/pixelGo/src/pix-demo/rout/leak.go:30 +0x10e
created by pixelgo/rout.JumpForSignal
    F:/code/pixelGo/src/pix-demo/rout/leak.go:28 +0x93
```
----
### 协程id
接下来我们来探索协程标识：**协程id**，在Go中，每个运行的协程都会分配一个协程id，一个常见的方式是从函数运行栈获取，引用之前网上其他玩家的写法：
```go
func main() {
    fmt.Println(getGID())
}

func getGID() uint64 {
    b := make([]byte, 64)
    b = b[:runtime.Stack(b, false)]
    b = bytes.TrimPrefix(b, []byte("goroutine "))
    b = b[:bytes.IndexByte(b, ' ')]
    n, _ := strconv.ParseUint(string(b), 10, 64)
    return n
}
```

我们来看看```runtime.stack()``` 会返回什么呢，其中真实内容是这样的：
```go
goroutine 21 [running]:
leaktest.interestingGoroutines(0xdb9980, 0xc00038e018, 0x0, 0x0, 0x0)
    F:/code/pixelGo/src/leaktest/leaktest.go:81 +0xbf
leaktest.CheckContext(0xdbe398, 0xc000108040, 0xdb9980, 0xc00038e018, 0x0)
    F:/code/pixelGo/src/leaktest/leaktest.go:141 +0x6e
leaktest.CheckTimeout(0xdb9980, 0xc00038e018, 0x3b9aca00, 0x0)
    F:/code/pixelGo/src/leaktest/leaktest.go:127 +0xe5
leaktest.TestCheck.func8(0xc000384780)
    F:/code/pixelGo/src/leaktest/leaktest_test.go:122 +0xaf
testing.tRunner(0xc000384780, 0xc000100050)
    D:/dev/go1.16/src/testing/testing.go:1193 +0x1a3
created by testing.(*T).Run
    D:/dev/go1.16/src/testing/testing.go:1238 +0x63c

goroutine 1 [chan receive]:
testing.(*T).Run(0xc000037080, 0xd8486a, 0x9, 0xd9ebc8, 0x304bd824304bd800)
    D:/dev/go1.16/src/testing/testing.go:1239 +0x66a
testing.runTests.func1(0xc000036f00)
    D:/dev/go1.16/src/testing/testing.go:1511 +0xbd
testing.tRunner(0xc000036f00, 0xc00008fc00)
    D:/dev/go1.16/src/testing/testing.go:1193 +0x1a3
testing.runTests(0xc0000040d8, 0xf40460, 0x5, 0x5, 0x0, 0x0, 0x0, 0x21cbf1c0100)
    D:/dev/go1.16/src/testing/testing.go:1509 +0x448
testing.(*M).Run(0xc0000c0000, 0x0)
    D:/dev/go1.16/src/testing/testing.go:1417 +0x514
main.main()
    _testmain.go:51 +0xc8
```
可以发现这个栈和我们运行panic抛出的信息非常类似，需要注意的是，++通过这种方式获取协程id并不是一个高效的方式。++  
实际生产使用过程并不提倡，值得一提的是，有时候日志框架又**需要我们打印出当前协程id，方便我们更好的定位问题上下文**。  

比如这是一个生产案例日志输出：
```go
// 1号协程用于初始化资源
[0224/162532.310:INFO:gid-1:yx_trace.go:66] cfg:&{ false false [] 0xc000295140 0xc0001d4e00 <nil> <nil> <nil>}
[0224/162532.320:INFO:gid-1:main.go:50] GameRoom Startup->
[0224/162532.320:INFO:gid-1:config_manager.go:107] configManager SetHttpListenAddr:8080
[0224/162532.320:INFO:gid-1:room_manager.go:57] roomManager Startup
[0224/162532.323:INFO:gid-1:room_manager.go:72] roomManager initPrx.
[0224/162532.330:INFO:gid-1:bootstrap.go:153] GameRoom START ok.
// 60号协程分配用于启动HTTP Server
[0224/162533.277:INFO:gid-60:expose.go:36] Start for HTTP server...
[0224/162533.277:INFO:gid-60:expose.go:39] register for debug server...
```
日志框架是力求对业务性能影响最低的，那么他是怎么获取协程id的呢？
### 汇编获取
通过协程绑定的g指针，具体可以参考[《Go高级编程》的做法](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-08-goroutine-id.html)

```go
// 记录各个版本的偏移量
var offsetDictMap = map[string]int64{
    "go1.12":    152,
    "go1.12.1":  152,
    "go1.12.2":  152,
    "go1.12.3":  152,
    "go1.12.4":  152,
    "go1.12.5":  152,
    "go1.12.6":  152,
    "go1.12.7":  152,
    "go1.13":    152,
    "go1.14":    152,
    "go1.16.12":    152,
}

// offset for go1.12
var goid_offset uintptr = 152
//go:nosplit
func getG() interface{}

func GoId() int64

// 部分汇编代码
// func getGptr() unsafe.Pointer
TEXT ·getGptr(SB), NOSPLIT, $0-8
    MOVQ (TLS), BX
    MOVQ BX, ret+0(FP)
    RET

TEXT ·GoId(SB),NOSPLIT,$0-8
    NO_LOCAL_POINTERS
    MOVQ ·goid_offset(SB),AX
    // get runtime.g
    MOVQ (TLS),BX
    ADDQ BX,AX
    MOVQ (AX),BX
    MOVQ BX,ret+0(FP)
    RET
```

### 性能比较：
我们来简单测试下两种获取go协程id方式性能差距：
```go
// BenchmarkGId-8       1000000000           0.0005081 ns/op
func BenchmarkGRtId(b *testing.B) {
    for n := 0; n < 1000000000; n++ {
        // runtime获取协程id
        getGID()
    }
}

// BenchmarkLog-8       1000000000           0.05731 ns/op
func BenchmarkGoId(b *testing.B) {
    for n := 0; n < 1000000000; n++ {
        // 汇编方式获取
        GoId()
    }
}
```
可以看到通过汇编方式获取协程id的方式性能更优，相差两个数量级。

## 参考链接
pprof介绍
- https://qcrao.com/2019/11/10/dive-into-go-pprof/  

协程泄露：
- https://www.storj.io/blog/finding-goroutine-leaks-in-tests  
- https://github.com/fortytw2/leaktest  

协程id：
- https://blog.sgmansfield.com/2015/12/goroutine-ids/