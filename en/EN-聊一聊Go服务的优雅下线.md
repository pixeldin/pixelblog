---
title: "Talk about Go: The gracefully handling of service shutdown"
date: 2021-11-02
<!-- thumbnailImagePosition: left -->
<!-- thumbnailImage: /img/go-context.jpg -->
<!-- thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/lastgreen.png -->
categories:
- Go
- microservice
tags:
- Talk about Go
- microservice
- HA
- process communication

metaAlignment: center
---

From high availability to graceful shutdown/restart, to refinement to program offline/restart, etc. So what can we do in Go? Today let's talk about the graceful shutdown of go programs.
<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/recover.png)

### Foreword
Recently, the importance of high availability(HA) of services has become more and more important. High availability usually refers to corresponding operations such as failover to redundant modules, such as active-standby switchover, to ensure that the system provides external availability. Today, let's talk about the graceful shutdown of Go programs. Is about how letting the program process old connections before shutdown or restart, and trying to achieve non-aware switching.
### Concept introduction
#### inter-process communication(IPC)
We know that there are several common ways of process communication:
- Pipeline
- Semaphores
- Socket
- Shared memory  

This time, let's talk about semaphores, such as P/V semaphores, It is often used for a process to wake up or wait for other processes in the critical section when it accesses the critical section. The semaphore is essentially an interrupt mechanism sent by the operating system. In addition to the P/V semaphore, there are common scenarios such as when we are interrupting by pressing ```Ctrl+C``` to notify the process to exit, actually it will send an interrupt signal, also called ```SIGINT```.  

In Go, the semaphore semantics on the windows platform are as follows:
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
Using 15 numbers to represent it in hexadecimal, then let's see how to monitor the system semaphore in go?  
```go
func Notify(c chan<- os.Signal, sig ...os.Signal) {
	if c == nil {
		panic("os/signal: Notify using nil channel")
	}
	// some code is omitted
    add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			if handlers.ref[n] == 0 {
				enableSignal(n)

				// listening in singleton, make sure for regist logic before the program start.
				watchSignalLoopOnce.Do(func() {
					if watchSignalLoop != nil {
						go watchSignalLoop()
					}
				})
			}
			handlers.ref[n]++
		}
	}
	// some code is omitted
}
```
The ```watchSignalLoop``` is a polling function in the unix version,
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
So far, we know the general process of semaphore registration and monitoring. By registering a context with the target semaphore, a coroutine is created asynchronously to monitor system signals.  

----
Next, let's take ```interrupt``` as an example to monitor the interrupt request of the system, which can be registered in Go as follows:
```go
// register a ctx bound to os.Interrupt
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)

//...

// unbind the context and semaphore
defer stop()
```
After listening to the context returned by ```os.Interrupt```, if the system call is interrupted, the ctx call ```ctx.Done()``` to be terminated, we can use this as our subsequent processing signal.

#### Graceful shutdown
After getting the interrupt semaphore, let's see how to exit gracefully. Let's take a look at this function
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
As you can see from the comments, the execution of ``Shutdown()`` will first close the open connection, then close the idle connection, and then wait for the used connection to become an idle connection before closing. Additionally, if the passed in ```ctx``` context expires before the shutdown is performed, ```Shutdown()``` will return an appropriate error.  

So we can use ```Shutdown()``` to let the program perform the final finishing work at the point of interruption, and use the life cycle of the context to control the buffer period of the ending.  

**Code example:**
```go
var (
    server http.Server
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	server = http.Server{
		Addr: ":8080",
	}

	// register route
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Second * 3)
		fmt.Fprint(w, "Hello World")
	})

	// start listening
	go server.ListenAndServe()

	// interrupt signal triggered
	<-ctx.Done()

	// unbind the ctx and the signal
	stop()
	log.Print("shutting down gracefully, press Ctrl+C again to force stop")

	// use 10 seconds to recycle the connection
	timeoutCtx, cancelFunc := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancelFunc()

	if err := server.Shutdown(timeoutCtx); err != nil {
		log.Print(err)
	}

	log.Print("graceful shutdown finished.")

}
```
1. Register a simple routing request, wait 3 seconds and return "Hello World"
2. Bind the system semaphore ```Signal.SIGNINT``` to the context
3. Interrupting through context awareness
4. Create a new context with a 10 second lifetime
5. Pass a context with a lifecycle to the ```Shutdown()``` function to control the end

**Output**  
Start the program and press ```Ctrl+C```, the program terminates quickly without request.
```bash
$ go run main.go
2021/11/06 23:58:03 shutting down gracefully, press Ctrl+C again to force stop
2021/11/06 23:58:03 graceful shutdown finished.
```
Then we call the request after the program starts
```bash
$ curl 127.0.0.1:8080
Hello World
```
and press ```Ctrl+C``` on the server side
```bash
$ go run main.go
2021/11/06 23:58:33 shutting down gracefully, press Ctrl+C again to force stop
2021/11/06 23:58:35 graceful shutdown finished.
```
You can see the log output, the program no longer exits immediately, but waits for the request to terminate before closing.  
And if we adjust the request execution logic to take longer, when the processing time exceeds the context period bound by the ```shutdown``` function, the program will return a context timeout error.  
```bash
2021/11/07 00:02:46 shutting down gracefully, press Ctrl+C again to force stop
2021/11/07 00:02:51 context deadline exceeded
```

#### Expand something
The above is the general implementation of graceful exit, about the idea of scalability:  
In the production scenario, there are other specific detection measures when the service is offline or unavailable. For example, **heartbeat timeout**, service offline in k8s can be judged by **monitoring a local file/handle** in the polling cycle, etc. In fact, the semaphore is only a way for us to perceive program interruption. line, we know that we can finally use ```Shutdown()``` to perform the finalization.  
What's more, after the execution is completed, if the context deadline exceeded is encountered, the business processing layer can generally archive the unprocessed requests, put them in the retry queue or record them in the form of writing logs down, as a follow-up repair credential.

### Reference link
[Graceful Shutdowns in Golang with signal.NotifyContext](https://millhouse.dev/posts/graceful-shutdowns-in-golang-with-signal-notify-context)  