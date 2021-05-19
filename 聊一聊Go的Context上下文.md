---
title: "聊一聊Go的Context上下文"
date: 2020-06-19
<!-- thumbnailImagePosition: left -->
<!-- thumbnailImage: /img/go-context.jpg -->
<!-- thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/go-context.jpg -->
categories:
- Go
- 后端
tags:
- Gin
showSocial: false
---

前面在“聊一聊http框架**httprouter**”的时候，提到了上下文的概念
<!--more-->

### 前言
前面在“聊一聊http框架**httprouter**”的时候，提到了上下文的概念，上一个demo用来列举web框架中全局变量的传值和设置，还类比了**Java Spring框架中**的```ApplicationContext```。  
这一次我们就来聊一聊Go中的标准库的context，梳理上下文概念在go中的常用情景。

### 问题引入
在列举上下文的用法之前，我们来看一个简单的示例：

#### 协程泄露
```go
func main()  {
	//打印已有协程数量
	fmt.Println("Start with goroutines num:", runtime.NumGoroutine())
	//新起子协程
	go Spawn()
	time.Sleep(time.Second)
	fmt.Println("Before finished, goroutines num:", runtime.NumGoroutine())
	fmt.Println("Main routines exit!")
}

func Spawn()  {
	count := 1
	for {
		time.Sleep(100 * time.Millisecond)
		count++
	}
}
```

**输出：**
```
Start with goroutines num: 1
Before finished, goroutines num: 2
Main routines exit!
```

我们在主协程创建一个子协程，利用```runtime.NumGoroutine()```打印当前协程数量，可以知道在**main协程**被唤醒之后退出前的那一刻，程序中仍然存在两个协程，可能我们已经习惯了这种现象，杀掉主协程的时候，退出会附带把子协程干掉，这种让子协程“自生自灭”的做法，其实是不太优雅的。 

### 解决方式
#### 管道通知退出
关于控制子协程的退出，可能有人会想到另一个做法，我们再来看另一个例子。  
```go
func main()  {
	defer fmt.Println("Main routines exit!")
	ExitBySignal()
	fmt.Println("Start with goroutines num:", runtime.NumGoroutine())
	//主动通知协程退出
	sig <- true
	fmt.Println("Before finished, goroutines num:", runtime.NumGoroutine())
}

//利用管道通知协程退出
func ListenWithSignal()  {
	count := 1
	for {
		select {
		//监听通知
		case <-sig:
			return
		default:
			//正常执行
			time.Sleep(100 * time.Millisecond)
			count++
		}
	}
}
```

**输出：**  
```
Start with goroutines num: 2
Before finished, goroutines num: 1
Main routines exit!
```
上面这个例子可以说相对优雅一些，在main协程里面主动通知子协程退出，不过两者之间的仍然存在依赖，假如子协程A又创建了新的协程B，这个时候通知只能到达子A，新协程B是无法感知的，因此同样可能会存在协程泄露的现象。


#### 上下文管理
下面我们会引进一个利用```Go```标准库context包的处理方式，通过上下文管理子协程的生命周期。

##### 官方概念  
在此之前，先来复习下标准库的概念：
> A Context carries a deadline, a cancellation signal, and other values across API boundaries.
Context's methods may be called by multiple goroutines simultaneously.  
上下文带着截止时间、cancel信号、还有在API之间提供值读写。

```go
type Context interface {
	// 返回该上下文的截止时间，如果没有设置截至时间，第二个值返回false
	Deadline() (deadline time.Time, ok bool)

	// 返回一个管道，上下文结束(cancel)时该方法会执行，经常结合select块监听
	Done() <-chan struct{}

	// 当Done()执行时，Err()会返回一个error解释退出原因
	Err() error

	// 上下文值存储字典
	Value(key interface{}) interface{}
}
```

其中较为常用的是```context.Withcancel()```函数，它会返回一个包装了```cancel()```函数的子上下文，当我们认为协程需要结束的时候，调用其返回值```cancel()```函数，子上下文会关闭内部封装的管道```Done()```来通知相应协程。

<!--```go-->
<!--// A canceler is a context type that can be canceled directly. The-->
<!--// implementations are *cancelCtx and *timerCtx.-->
<!--type canceler interface {-->
<!--	cancel(removeFromParent bool, err error)-->
<!--	//终止管道，关闭即退出-->
<!--	Done() <-chan struct{}-->
<!--}-->
<!--```-->

```go
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    //返回新的cancelCtx子上下文
    c := newCancelCtx(parent)
    //将原上下文
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

我们继续引入一个协程泄漏的demo，来看下具体使用例子：  
**使用示例：**  
```go
//阻塞两个子协程，预期只有一个协程会正常退出
func LeakSomeRoutine() int {
	ch := make(chan int)
	//起3个协程抢着输入到ch
	go func() {
		ch <- 1
	}()

	go func() {
		ch <- 2
	}()

	go func() {
		ch <- 3
	}()
	//一有输入立刻返回
	return <-ch
}

func main() {
	//每一层循环泄漏两个协程
	for i := 0; i < 4; i++ {
		LeakSomeRoutine()
		fmt.Printf("#Goroutines in roop end: %d.\n", runtime.NumGoroutine())
	}
}
```

**程序输出：**  
```
#Goroutines in roop end: 3.
#Goroutines in roop end: 5.
#Goroutines in roop end: 7.
#Goroutines in roop end: 9.
```
可以看到，随着循环次数增加，除去main协程，每一轮都**泄漏两个协程**，所以程序退出之前最终有9个协程。  

接下来我们引入上下文的概念，来管理子协程：  
```go
func FixLeakingByContex() {
	//创建上下文用于管理子协程
	ctx, cancel := context.WithCancel(context.Background())

	//结束前清理未结束协程
	defer cancel()

	ch := make(chan int)
	go CancelByContext(ctx, ch)
	go CancelByContext(ctx, ch)
	go CancelByContext(ctx, ch)

	// 随机触发某个子协程退出
	ch <- 1
}

func CancelByContext(ctx context.Context, ch chan (int)) int {
	select {
	case <-ctx.Done():
		//fmt.Println("cancel by ctx.")
		return 0
	case n := <-ch :
		return n
	}
}

func main() {
	//每一层循环泄漏两个协程
	for i := 0; i < 4; i++ {
		FixLeakingByContex()
		//给它点时间 异步清理协程
		time.Sleep(100)
		fmt.Printf("#Goroutines in roop end: %d.\n", runtime.NumGoroutine())
	}
}
```
**程序分析：**  
可以看到```CancelByContext```函数对管道进行轮询，程序只有两个分支方可return
- 上下文通知结束
- 收到管道传入的值

我们在``FixLeakingByContex()``函数结束前defer了```cancel()```函数，因此会在程序退出前把相应的子协程**clean**掉，所以可以看到如下输出，每一轮都只剩下一个**main**协程。

**程序输出：**  
```
#Goroutines in roop end: 1.
#Goroutines in roop end: 1.
#Goroutines in roop end: 1.
#Goroutines in roop end: 1.
```

#### 截止退出
除了我们手动调用cancel函数退出之外，标准库还提供了两个限时退出的操作，```WithDeadline(parent Context, d time.Time)```以及```WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)```函数，可以传入一个时间点或者时间段，表示经过该时间之后自动调用该上下文的```Done()```函数。

**使用示例：**  
我们摘取官网一个demo：  
```go
func main() {
	// 由于传入时间为50微妙，因此在select选择块中，ctx.Done()分支会先执行
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err()) // prints "context deadline exceeded"
	}
}
//输出： context deadline exceeded
```

----

#### 请求超时
关于上下文```WithTimeout()```函数的用法，还可以判断一个```http```请求是否超时。  

**程序示例：**  
```go
func TestTimeReqWithContext(t *testing.T) {
	//初始化http请求
	request, e := http.NewRequest("GET", "https://www.pixelpigpigpig.xyz", nil)
	if e != nil {
		log.Println("Error: ", e)
		return
	}

	//使用Request生成子上下文, 并且设置截止时间为10毫秒
	ctx, cancelFunc := context.WithTimeout(request.Context(), 10*time.Millisecond)
	defer cancelFunc()

	//绑定超时上下文到这个请求
	request = request.WithContext(ctx)

	//time.Sleep(20 * time.Millisecond)

	//发起请求
	response, e := http.DefaultClient.Do(request)
	if e != nil {
		log.Println("Error: ", e)
		return
	}

	defer response.Body.Close()
	//如果请求没问题, 打印body到控制台
	io.Copy(os.Stdout, response.Body)

}
```
我们给请求限时10毫秒，执行可以看到程序打印上下文已经过期了。  
**输出示例：**
```
=== RUN   TestTimeReqWithContext
2020/05/16 23:17:14 Error:  Get https://www.pixelpigpigpig.xyz: context deadline exceeded
```

----

### Context值读写
如前面所示， ```context```还提供了一个读写的函数签名：
```go
// 上下文值存储字典
Value(key interface{}) interface{}
```
关于它的用法，之前在“聊一聊httpRouter”的文章中列举过，可以这样子使用，  

**示例：**  
```go
// 获取顶级上下文
ctx := context.Background()
// 在上下文写入string值, 注意需要返回新的value上下文
valueCtx := context.WithValue(ctx, "hello", "pixel")
value := valueCtx.Value("hello")
if value != nil {
	fmt.Printf("Params type: %v, value: %v.\n", reflect.TypeOf(value), value)
}
```

```context.Background()```是一个非空的顶级上下文，只要程序还在它就不会取消，也没有嵌入值，经常被main函数所使用。  
关于```context```的值读取，在Go规范中有一个约定，不应使用它来封装寿命长期的参数，一般仅用于传输一个请求作用域的值，关于程序的业务参数应该暴露出来，放在函数参数列表中，以提高可读性。  

----


### Context的作用域
上面列举的```Context```几个用法，可以说比较常见，前面曾经拿```go```中```context```的来类比**Java Spring**中的```applicationContext```全局上下文，但是严格来说，其实这个是带有争议的。因为```Spring```的上下文是贯穿整个程序的生命周期，往往会附带一些全局设置项，在```go```中，比较倾向于用于控制一个子程序块，和**Spring**的全局上下文比较，```Go```的```context```是比较短暂的。

Go里面```context.Context```的常用情景：
- 用于贯穿一个子协程的任务片段，如上面用于把控子协程的退出
- 或者是在网络框架中表示一个请求，管理其开始至结束，如在```Gin```框架中的从```Request```到```Response```，并不是全局的。


另外在参考链接处有一篇比较有意思的争论，有个作者**吐槽了关于Go上下文的泛滥**，关于上下文的辩论在该文章的评论可谓见仁见智。  

> Use context values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions  

总的来说，**Go上下文应该着重用于管理子协程或者是有目地嵌入函数链的生命周期中，而不是约定俗成在每个方法都传播上下文。**


#### 参考链接
**Goroutine leak**  
https://medium.com/golangspec/goroutine-leak-400063aef468  
**Understanding the context package**  
https://medium.com/rungo/understanding-the-context-package-b2e407a9cdae  
**Context Package Semantics In Go**  
https://www.ardanlabs.com/blog/2019/09/context-package-semantics-in-go.html  
**Context should go away for Go 2**(“关于Go上下文泛滥的吐槽”)  
https://faiface.github.io/post/context-should-go-away-go2/