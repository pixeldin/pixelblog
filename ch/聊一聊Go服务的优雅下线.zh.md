---
title: "聊一聊Go服务的优雅处理(一)--优雅下线"
date: 2021-11-02
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/lastgreen.png
categories:
- Go
- HA
tags:
- 服务治理
- 进程通信
- 优雅关闭

metaAlignment: center
---

从高可用到优雅关闭/重启，细化到程序下线/重启等操作，在Go里面有哪些处理方式呢？今天我们来聊聊go程序的优雅关闭
<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/recover.png)

### 前言
最近服务高可用的重要性越来越大，高可用通常指的是通过故障转移到冗余模块，如主备切换等相应操作，用来保证系统对外提供可用性，而细化到程序下线/重启等操作，在Go里面有哪些处理方式呢？今天我们来聊聊Go程序的优雅关闭与重启，如何让程序在关闭或者重启之前对旧的连接进行处理，尽量做到无感知切换。
### 概念引入
#### 进程间通讯方式
我们知道进程通信有几种常用的方式：
- 管道
- 信号量
- 网络socket
- 共享内存  

今天我们先来聊一聊信号量，比如P/V信号量，常常用于进程在访问临界区时候，用于唤醒或等待临界区的其他进程，信号量本质上是操作系统发送的一个中断机制，除了P/V信号量，还有常见的场景比如我们在中断按下```Ctrl+C```用于通知进程退出，会发送一个interrupt信号，也叫SIGINT。  

在Go里面，windows平台下的信号量语义如下：
```go
const (
	// More invented values for signals
	SIGHUP  = Signal(0x1)
	SIGINT  = Signal(0x2)
	SIGQUIT = Signal(0x3)
	SIGILL  = Signal(0x4)
	SIGTRAP = Signal(0x5)
	SIGABRT = Signal(0x6)
	SIGBUS  = Signal(0x7)
	SIGFPE  = Signal(0x8)
	SIGKILL = Signal(0x9)
	SIGSEGV = Signal(0xb)
	SIGPIPE = Signal(0xd)
	SIGALRM = Signal(0xe)
	SIGTERM = Signal(0xf)
)

var signals = [...]string{
	1:  "hangup",
	2:  "interrupt",
	3:  "quit",
	4:  "illegal instruction",
	5:  "trace/breakpoint trap",
	6:  "aborted",
	7:  "bus error",
	8:  "floating point exception",
	9:  "killed",
	10: "user defined signal 1",
	11: "segmentation fault",
	12: "user defined signal 2",
	13: "broken pipe",
	14: "alarm clock",
	15: "terminated",
}
```
使用15个数字以十六进制表示，那么我们接着看，在go里面，是怎么监听系统信号量的呢？  
```go
func Notify(c chan<- os.Signal, sig ...os.Signal) {
	if c == nil {
		panic("os/signal: Notify using nil channel")
	}
	// 省略部分代码...
    add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			if handlers.ref[n] == 0 {
				enableSignal(n)

				// 单例启动监听，保证程序启动之前注册相应的处理逻辑
				watchSignalLoopOnce.Do(func() {
					if watchSignalLoop != nil {
					    // 新建协程轮询监听
						go watchSignalLoop()
					}
				})
			}
			handlers.ref[n]++
		}
	}
	// 省略部分代码...
}
```
其中的```watchSignalLoop```在unix版本中，是一个轮询函数，
```go
func loop() {
	for {
		process(syscall.Signal(signal_recv()))
	}
}

func init() {
	watchSignalLoop = loop
}
```
至此我们知道了信号量注册和监听的大致过程了，通过注册一个与目标信号量的上下文，异步创建一个协程进行系统信号监听。  

----
接下来我们拿```interrupt```来举例，监听系统的中断请求，在Go中可以用如下方式注册：
```go
// 注册返回绑定了os.Interrupt的ctx
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)

//...

// 其中stop()函数用于解绑上下文与信号量
defer stop()
```
通过监听```os.Interrupt```返回的上下文之后，如果系统调用中断，该ctx会执行终止，也就是```ctx.Done()```，我们可以利用这个作为我们后续处理的信号量。

#### 优雅关闭
拿到中断信号量之后，我们来看下如何优雅退出，来看下这个函数
```go
// Shutdown gracefully shuts down the server without interrupting any
// active connections. Shutdown works by first closing all open
// listeners, then closing all idle connections, and then waiting
// indefinitely for connections to return to idle and then shut down.
// If the provided context expires before the shutdown is complete,
// Shutdown returns the context's error, otherwise it returns any
// error returned from closing the Server's underlying Listener(s).
func (srv *Server) Shutdown(ctx context.Context) error {
    // ...
}
```
从注释可以看到，```Shutdown()```执行会先关闭打开连接，然后关闭空闲连接，接着等待已使用连接变成空闲连接，才会执行关闭。此外，如果传入的```ctx```上下文在执行关闭前发生过期，则```Shutdown()```会返回相应错误。  

所以我们可以利用```Shutdown()```，让程序在中断处，执行最后收尾工作，另外用上下文的生命周期来把控收尾的缓冲期。

**代码示例:**
```go
var (
    server http.Server
)

// 优雅停止demo
func main() {
	// 注册返回绑定了os.Interrupt的ctx
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	server = http.Server{
		Addr: ":8080",
	}

	// 注册路由
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Second * 3)
		fmt.Fprint(w, "Hello World")
	})

	// 启动监听
	go server.ListenAndServe()

	// 触发interrupt信号
	<-ctx.Done()

	// 解绑上下文与信号量
	stop()
	fmt.Println("shutting down gracefully, press Ctrl+C again to force")

	// 最后10秒回收连接
	timeoutCtx, cancelFunc := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancelFunc()

	if err := server.Shutdown(timeoutCtx); err != nil {
		fmt.Println(err)
	}

}
```
1. 注册一个简单的路由请求，等待3秒之后返回“Hello World”
2. 绑定系统信号量```Signal.SIGNINT```到上下文
3. 通过上下文感知中断
4. 新建10秒生存期的上下文
5. 传入带生命周期的上下文至```Shutdown()```函数，用于控制收尾

**输出示例**  
启动程序并且按下```Ctrl+C```，在没有请求的情况下，程序快速终止。
```bash
$ go run main.go
2021/11/06 23:58:03 接收到SIGINT信号, 执行优雅停止, 等待收尾...
2021/11/06 23:58:03 程序关闭完成.
```
接着我们在程序启动之后执行请求让其耗时处理
```bash
$ curl 127.0.0.1:8080
Hello World
```
并在服务端按下```Ctrl+C```
```bash
$ go run main.go
2021/11/06 23:58:33 接收到SIGINT信号, 执行优雅停止, 等待收尾...
2021/11/06 23:58:35 程序关闭完成.
```
可以看到日志输出，程序不再是立即退出，而是等待请求终止才会关闭。  
而假如说我们调整请求执行逻辑耗时更长，当处理时长超过```shutdown```函数绑定的上下文周期，则程序会返回一个上下文超时的错误。  
```bash
2021/11/07 00:02:46 接收到SIGINT信号, 执行优雅停止, 等待收尾...
2021/11/07 00:02:51 优雅停止错误: context deadline exceeded
```

#### 抛砖引玉
以上就是优雅退出的大致实现，关于可拓展的想法：  
上述主要是一个优雅下线之前的处理，生产场景下，服务下线或者不可用还有其他的具体检测措施，比如**心跳包超时丢失**，k8s中服务下线可以通过轮询周期**监听一个本地文件/句柄**来判断等，其实信号量只是我们感知程序中断的一种方式，基于服务下线，我们知道了最终可以使用```Shutdown()```来执行收尾。  
此外，当执行收尾之后，如果遇到关联上下文已经超时的情况```context deadline exceeded```，业务处理层一般可以归档未处理完成的请求，放入重试队列或者以写日志的形式记录下来，作为后续修复凭据。

### 参考链接
[Graceful Shutdowns in Golang with signal.NotifyContext](https://millhouse.dev/posts/graceful-shutdowns-in-golang-with-signal-notify-context)  