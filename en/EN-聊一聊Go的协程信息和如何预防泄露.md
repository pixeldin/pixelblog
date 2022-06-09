---
title: "Talk about Go: Goroutine's information and how to prevent goroutine leaks"
date: 2022-03-17
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/routine.PNG
categories:
- Go
- Goroutine
tags:
- Talk about Go

metaAlignment: center
---

Goroutines are a common concept in Go, which starts and ends with the life cycle of a go program. Today, let's talk about Go's goroutine leak.  
<!--more-->

![协程通信](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/road.PNG)

## Foreword  
About the underlying concept of goroutine, GMP model, etc. It has been analyzed in the previous blog.   
If you are interested, you can read the previous article [Talk about Go: The operating system's thread scheduling and Goroutines](/en/2020/10/talk-about-go-the-operating-systems-thread-scheduling-and-goroutines/)

## Leak case
The reason why goroutine leaks are common is that sometimes we ignore the problem until the resource load is abnormal.
When excluding exceptions in the production environment before, I have encountered the scenario of memory leakage of the go program.   

Memory leakage is closely related to goroutine leakage, which is essentially caused by non-recycling of resources. Here is a brief restoration of the scene, the details have been deleted, and the approximate form is as follows:

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
    // Return as soon as there is input
    return <-ch
}

func main() {
    // ...
    JumpForSignal()
    // ...
}
```

After analyzing the demo, we can see that this function call will block two sub-coroutines, and only one goroutine is expected to exit normally.

## Get goroutine information
Since there is a goroutine leak, how can we avoid or discover it in our daily work? Below we list a few ideas.
### Follow the guidelines
Since Go is a language with its own GC, many times when writing code, you do not need to care about the resource release of variables, unlike C programmers who need to release variables at the end after they apply.   

However, Go's ```chan``` has some guidelines when it is used. When it is determined that chan is no longer used, it can be closed on the output side to avoid other coroutines still waiting the ```chan``` for output.

### Number of coroutines
To find the leaked goroutine, the first thing you can think of is the number of goroutines. When your function processing logic is relatively simple, except for the main goroutine, the expected coroutines should all return before the end, which can be called at the end of the main function. ```runtime``` package functions:
```go
// NumGoroutine returns the number of goroutines that currently exist.
func NumGoroutine() int {
    return int(gcount())
}
```
We can return the total number of current coroutines by this:
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
Output：
```bash
Number of goroutines:1
Number of goroutines:3
```

### Goroutine function stack
There is also a relatively common form of locating coroutines. In Go, it can be used to analyze the context of goroutine functions. Common ones, such as the pprof that comes with go, are also obtained in this way. In actual cases, conditions can be enabled. **pprof** facilitates analysis.    

Let's take a look at an example. We add a ```http``` port listener to the above example to access the **pprof** analysis tool that comes with go.  

Then enter in the browser:  
```go
http://localhost:8899/debug/pprof/goroutine?debug=1
```
You can get a list of coroutines for the entire program: 
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
The conclusion is: the current program has a total of 7 coroutines, it can be seen that there are 2 goroutine allocated in ```F:/code/pixelGo/src/pix-demo/leak/leak.go:24``` and ```F:/code/pixelGo/src/pix-demo/leak/leak.go:28```.

Sometimes it can also be analyzed in multiple dimensions, such as :
```go
http://localhost:8899/debug/pprof/goroutine?debug=2
```
You can see the different states of the current goroutine through the labels behind the goroutine, **running/io wait/chan send**
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
### Goroutine Id
Next, let's explore the goroutine identifier: **goroutine id**. In Go, each running goroutine will be assigned a goroutine id. A common way is to obtain it from the function running stack and refer to other students on the Internet. is written as:  
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

Let's see what ```runtime.stack()``` will return. The real content is this:
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

It can be found that this stack is very similar to the information thrown by us when we run panic. It should be noted that obtains the goroutine id in this way is not an efficient way. 

The actual production process is not recommended. It is worth mentioning that in order to facilitate us to better locate the problem context, sometimes the logging framework requires us to print out the current goroutine id.  

For example this is a production case log output:
```go
// gid-1 initialize resources
[0224/162532.310:INFO:gid-1:yx_trace.go:66] cfg:&{ false false [] 0xc000295140 0xc0001d4e00 <nil> <nil> <nil>}
[0224/162532.320:INFO:gid-1:main.go:50] GameRoom Startup->
[0224/162532.320:INFO:gid-1:config_manager.go:107] configManager SetHttpListenAddr:8080
[0224/162532.320:INFO:gid-1:room_manager.go:57] roomManager Startup
[0224/162532.323:INFO:gid-1:room_manager.go:72] roomManager initPrx.
[0224/162532.330:INFO:gid-1:bootstrap.go:153] GameRoom START ok.
// gid-60 assigned to start HTTP Server
[0224/162533.277:INFO:gid-60:expose.go:36] Start for HTTP server...
[0224/162533.277:INFO:gid-60:expose.go:39] register for debug server...
```
Often the logging framework strives to have the lowest impact on business performance. Since there are performance concerns, how does it obtain the goroutine id?  
In fact, in Go, there is a g pointer in the system thread structure bound to each goroutine. After getting the information of the g pointer, according to the offset of the g pointer structure (note that different go versions may be different ) to specify the fetch id.
### Compilation Method
The G pointer bound by the goroutine is here for reference[《Go高级编程》](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-08-goroutine-id.html)

```go
// Record the offset of each version
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

// part of the assembly code
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

### Performance comparison：
Let's simply test the performance gap between the two methods of obtaining the go goroutine id：
```go
// BenchmarkGRtId-8     1000000000           0.0005081 ns/op
func BenchmarkGRtId(b *testing.B) {
    for n := 0; n < 1000000000; n++ {
        // runtime get goroutine id
        getGID()
    }
}

// BenchmarkGoId-8      1000000000           0.05731 ns/op
func BenchmarkGoId(b *testing.B) {
    for n := 0; n < 1000000000; n++ {
        // get it by assembly
        GoId()
    }
}
```
It can be seen that the way to obtain the goroutine id by assembly is better, and the difference is several orders of magnitude.

----

## Limit coroutines
The above lists several methods for locating goroutine information. Is there any other way to control the program's go goroutine before the goroutine leaks? One way is to use a powerful channel to sit down restrictions.

** ### Step by st**ep
Here is a simple idea, that is, wrap a layer of ```channel``` for protection,
```go
// limited quantity
var LIMIT_G_NUM = make(chan struct{}, 100)

// custom processing logic
type HandleFun func()

func AsyncGoForHandle(fn HandleFun)  {
    // mark as handling
    LIMIT_G_NUM <- struct{}{}
    go func() {
        defer func() {
            if err := recover(); err != nil {
                log.Fatalf("AsyncGoForHandle recover from err: %v", err)
            }
            // return token
            <-LIMIT_G_NUM
        }()

        // processing logic function
        fn()
    }()
}
```
The above idea is relatively simple, I believe everyone can understand it. Every time you need to create a goroutine asynchronously, you only need to call the ```AsyncGoForHandle()``` function. The disadvantage may be the processing logic ```HandleFun()``` Not general enough, you need to define your own specific implementation.

Another way is to introduce the concept of **goroutine pool**. The pool here is a bit similar to the database connection pool, that is, it is pre-created at the beginning. As long as the business layer is responsible for submitting data, there are already many mature packages in the industry.

### Reliable Solution：tunny
I saw that the community has a well-packaged goroutine pool named **tunny**. The number of lines of code is not many. Let's try to disassemble and analyze the code. The project address:https://github.com/Jeffail/tunny

**Step 1: Define the processing logic interface**  
```go
type Worker interface {
    // Custom logic implementation, developers 
    // only need to care about input and output parameters
    Process(interface{}) interface{}
}
```

**Step 2: the input source of the wrapper worker workRequest**  
```go
type workerWrapper struct {
    // inject internal implementation logic
    worker        Worker
    interruptChan chan struct{}

    // request source
    reqChan chan<- workRequest

    // ...
}
```

**Step 3: input source structure**  
```go
type workRequest struct {
    // input
    jobChan chan<- interface{}

    // handle result, the return of worker.Process()
    retChan <-chan interface{}

    // ...
}
```

**Step 4: implement**  
As we know, Go's interfaces follow the duck model: as long as it behaves like a duck, it's a duck.
```go
// Worker implement
type closureWorker struct {
    processor func(interface{}) interface{}
}

func (w *closureWorker) Process(payload interface{}) interface{} {
    return w.processor(payload)
}
```

**Step 5: Define Work Pool Structure**
```go
type Pool struct {
    queuedJobs int64

    // member functions for duck entities
    ctor    func() Worker
    workers []*workerWrapper
    reqChan chan workRequest

    workerMut sync.Mutex
}

func NewFunc(n int, f func(interface{}) interface{}) *Pool {
    return New(n, func() Worker {
        return &closureWorker{
            // a real implement struct
            processor: f,
        }
    })
}

func New(n int, ctor func() Worker) *Pool {
    p := &Pool{
        ctor:    ctor,
        reqChan: make(chan workRequest),
    }
    // create coroutines in batches, monitor and process tasks from 'reqChan'
    p.SetSize(n)

    return p
}
```
The relevant entity structure is as follows, which is clearer when reading the source code.
![实体模块](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/sync-pool/en-job-pool.png)

This framework creating a pool of coroutines in advance, and then the business layer only needs to continuously input "processing data" into the chan of ```workRequest```, that is, ```process()```function, ```process()``` module will input data to the internal ```channel``` for processing, and the ```worker``` in the pool will process it.    
This factory pattern is worth learning from, and many frameworks in Go use this writing method.  

Quoting the usage example of the original project README.md:
```go
numCPUs := runtime.NumCPU()

pool := tunny.NewFunc(numCPUs, func(payload interface{}) interface{} {
    var result []byte    
    result = wrapSomething()
    return result
})
defer pool.Close()

http.HandleFunc("/work", func(w http.ResponseWriter, r *http.Request) {
    input, err := ioutil.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Internal error", http.StatusInternalServerError)
    }
    defer r.Body.Close()

    result := pool.Process(input)

    w.Write(result.([]byte))
})

http.ListenAndServe(":8080", nil)
```

## Summary
- Goutines have several built-in information, goroutine id, goroutine stack, goroutine status (running/io wait/chan send), which can help us avoid or locate problems to a certain extent.
- Only one Go keyword is required to create a goroutine in Go, but it is critical to recycle it reasonably. If necessary, the goroutine pool can be used as a limit

## Reference link
Introduce of pprof 
- https://qcrao.com/2019/11/10/dive-into-go-pprof/  

Goroutine Leaking：
- https://www.storj.io/blog/finding-goroutine-leaks-in-tests  
- https://github.com/fortytw2/leaktest  

Goroutine pool：
- https://github.com/Jeffail/tunny