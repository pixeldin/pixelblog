---
title: "Talk about Go: The gracefully handling of service restart"
date: 2021-12-23
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/lastgreen.png
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

Last time we talked about graceful shutdown of Go programs, today we will talk about graceful restarts.
<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/recover.png)

### Foreword
A while ago, we talked about how to close the Go program gracefully. I remembered that I saw an implementation of semaphore interaction before. I found it very interesting, so I took it out and share something of my opinion. The source article link will be placed in the reference link.  

In fact, the core of graceful restart is that we need to have a **Panman**, if the offline service has unfinished connections, we need to provide a new service/process to handle as much as possible, and continue to continuously monitor new requests, provide external availability, and make the requester unaware.

In short, achieving graceful restart requires solving two problems:
1. How to keep the originally created socket at the operating system level so that the newly restarted process can continue to listen
2. ensure that all subsequent requests will be able to respond or time out


This sounds ideal, so let's see how it is achieved step by step.

#### Key Step
1. Under the process currently listening to the socket, fork a child process to handle
2. New(child) process replace and reuse the original socket
3. New(child) process notify old(parents) process stop handle new request and close gradually

#### State Transition
In the early stage, we didn't go too deep into the details of the subsequent processing of the process startup. First, let's sort out the state that the program needs to monitor, or what events the program needs to respond to when it restarts.  
In fact, the current service is nothing more than two states,  
- State 1: the first time of starting
- State 2: starts a new process to replace the old process, when something happened like service upgate or version change

State 1 is actually not fundamentally different from ordinary services, that is, just start listening.   

Let's talk about State 2. State 2 is actually extended from State 1, so the program needs to monitor both states at the same time, and the trigger event of the monitoring is the ** semaphore** mentioned above in graceful stop.

I drew a general flow chart to facilitate further understanding:
![state change](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/en-graceful%20state%20change.png)

#### Pre-concept
Let's familiarize ourselves with some concepts in network socket programming so as to know how to reuse connections.  

We know that in a network environment, an end-to-end connection can be established using **TCP quads**, namely ```<src addr>, <src port>, <dest addr>, <dest port>``` to mark as the unique connection.  

A TCP connection disconnection needs to go through four waves, among which there is a ```TIME_WAIT``` state on the disconnected party, which is used to wait for the disconnected party to close the connection, or the sender buffer data is actually sent. This waiting time generally does not change (default 2min), which means that the current tcp tuple cannot be reused before the ```TIME_WAIT``` state ends, unless ```SO_REUSEADDR``` is set.  

There are two key parameters here, ```SO_REUSEADDR``` and ```SO_REUSEPORT```,
First of all, it literally means **reuse**. The specific concepts are as follows:

Parameter | Meaning
---|---
SO_REUSEADDR | Allow connection ip addresses to be reused without being completely disconnected
SO_REUSEPORT | Under the premise of enabling `SO_REUSEADDR`, the connection port address is allowed to be reused

Then if multiplexing is enabled and the connection is successful, at the operating system level, if multiple file handles are bound to the system's **ip+port**, what will the system do? The answer is load balancing, that is, the system will respond to requests. Assignment, similar to random polling, interacts with connections of the same unique ip and port.  

Some people may say here, whether there is a problem of out-of-bounds access to different client processes in this way, there is indeed a problem, so there is a convention based on security considerations:
> To prevent "port hijacking", there is one special limitation: All sockets that want to share the same address and port combination must belong to processes that share the same effective user ID

### Program example
Let's see how the program is implemented  

1. Configuration items for incoming multiplexed connections
```go
func control(network, address string, c syscall.RawConn) error {
	var err error
	c.Control(func(fd uintptr) {
		err = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEADDR, 1)
		if err != nil {
			return
		}
		err = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
		if err != nil {
			return
		}
	})
	return err
}
```

2. Check if the currently listening tcp tuple is listening
```go
func listener() (net.Listener, error) {
	lc := net.ListenConfig{
		Control: control,
	}
	if l, err := lc.Listen(context.TODO(), "tcp", ":8080"); err != nil {
	    // port not in use, return err
		return nil, err
	} else {
		return l, nil
	}
}
```

3. Monitor system semaphores
```go
func upgradeLoop(l *net.Listener, s *http.Server) {
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGQUIT, syscall.SIGUSR2)
	for t := range sig {
		switch t {
		case syscall.SIGUSR2:
			// receive upgrade semaphore
			log.Println("Received SIGUSR2 upgrading binary")
			// graceful upgrade of frok child processes
			if err := spawnChild(); err != nil {
				log.Println(
					"Cannot perform binary upgrade, when starting process: ",
					err.Error(),
				)
				continue
			}
		case syscall.SIGQUIT:
		    // receive a semaphore to kill the current process
			s.Shutdown(context.Background())
			os.Exit(0)
		}
	}
}

// fork child process, and update override global parent process id before new process
func spawnChild() error {
	// get the parameters of the current startup incoming executable file, such as./main
	argv0, err := exec.LookPath(os.Args[0])
	if err != nil {
		return err
	}
	wd, err := os.Getwd()
	if err != nil {
		return err
	}

	files := make([]*os.File, 0)
	files = append(files, os.Stdin, os.Stdout, os.Stderr)

	// save the current process, this id will be killed after the new process is started
	ppid := os.Getpid()
	os.Setenv("APP_PPID", strconv.Itoa(ppid))

	os.StartProcess(argv0, os.Args, &os.ProcAttr{
		Dir:   wd,
		Env:   os.Environ(),
		Files: files,
		Sys:   &syscall.SysProcAttr{},
	})

	return nil
}
```
4. Main coroutine
```go
func main() {
	log.Println("Started HTTP API, PID: ", os.Getpid())
	var l net.Listener
    // start at first
    if fd, err := listener(); err != nil {
    	log.Println("Parent does not exists, starting a normal way")
    	l, err = net.Listen("tcp", ":8080")
    
    	if err != nil {
    		panic(err)
    	}
    } else {
    	// the current port has been monitored
    	l = fd
    	// send quit to the parent process
    	killParent()
    	time.Sleep(time.Second)
    }
    
    // start server listening
    s := &http.Server{}
    	http.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
    		log.Printf("New request! From: %d, path: %s, method: %s: ", os.Getpid(),
    			r.URL, r.Method)
    	})
    go s.Serve(l)
    
    // monitor semaphore
	upgradeLoop(&l, s)
}
```

#### Code Reference：
[zero-downtime-application source code](https://github.com/wutchzone/zero-downtime-application/blob/master/inherit-version/main.go), in fact, the core lies in the process of elegant migration from new process to old process, as long as you understand the core step, it will look more clearer at the code.  

This is why the main function logic block needs to be compatible with two situations, one is the normal server process, and the other is to receive the end of the old process.


### Expand the application
Regarding the graceful restart triggering mechanism above, the user sends the semaphore ```pkill -SIGUSR2``` to the process, as a seamless switch for manual upgrade.  

In fact, based on this function, it can be expanded, such as adding connection detection to monitoring services, requesting response time alarms, etc. When a certain trigger mechanism is reached, graceful restart can be triggered, so as to achieve the effect of dynamic pull-up. Of course, the follow-up still needs to review the positioning service. After all sometimes rebooting doesn't solve everything.

### Reference link
- [Zero downtime API in Golang](https://wutch.medium.com/zero-downtime-api-in-golang-d5b6a52cc0ed)
- [Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)
- [基于K8S的优雅关闭](https://medium.com/over-engineering/graceful-shutdown-with-go-http-servers-and-kubernetes-rolling-updates-6697e7db17cf)
- [关于BSD套接字参数讨论](https://stackoverflow.com/questions/14388706/how-do-so-reuseaddr-and-so-reuseport-differ/14388707#14388707)