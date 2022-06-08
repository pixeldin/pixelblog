---
title: "聊一聊Go服务的优雅处理(二)--优雅重启"
date: 2021-12-23
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/restart.png
categories:
- Go
- HA
tags:
- 服务治理
- 进程通信
- 优雅重启

metaAlignment: center
---

上一次我们聊到Go程序如何优雅关闭，今天我们来聊一聊优雅重启。
<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/recover.png)

### 前言
前阵子我们聊到Go程序如何优雅关闭，想起之前看到一篇信号量交互的实现，个人觉得挺有意思，所以拿出来梳理一下，文章连接会放在参考资料。  

其实优雅重启核心在于我们需要有一个**接盘侠**，当下线的服务如果有未处理完的连接，我们需要提供一个新的服务/进程尽可能地处理，并继续持续监听新的请求，对外提供可用性，让请求端无感知。 

简单来说，实现优雅重启需要解决两个问题：
1. 如何在操作系统层面，保留原先创建的socket让新重启的进程继续监听
2. 保证所有后续请求能够执行响应或者超时


这听起来似乎十分理想，下面我们一步一步拆解，看下是如何实现的。

#### 核心拆解
1. 在当前监听socket的进程下，fork一个子进程进行“接盘”
2. 新(子)进程接替，复用原先的socket
3. 新(子)进程通知原(父)进程停止接收请求并关闭

#### 状态转移
我们前期不过多深入进程启动后续处理的细节，先来梳理下程序需要监听的状态，或者说程序在重启时刻需要对哪些事件做出什么响应。
其实当前服务无非两个状态，
- 一个是首次启动
- 另一个是版本变动启动新进程替换旧进程  

状态一其实和普通的服务没有本质区别，就是启动完进行listen就好了。   

来聊一聊状态二，状态二其实是由状态一延伸出来的，所以程序需要同时兼任两种状态的监听，而监听的触发事件就是上文我们在优雅停止中提到的**信号量**。

我画了一张大致流程图，方便后续加深理解：
![状态转移](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/graceful/graceful-state.png)

#### 前置概念
我们再来熟悉下网络Socket编程中一些概念，以便知悉如何进行连接复用。

我们知道在网络环境中，可以使用**TCP四元组**建立一个端到端的连接，即```<src addr>, <src port>, <dest addr>, <dest port>```锁定唯一连接标识。  

都知道一个TCP连接断开需要经过四次挥手，其中在被断开方有个```TIME_WAIT```状态，用于等待被断开方关闭连接，或者是发送端缓冲区数据真正发送，这个等待时间一般是不会改变的(默认2min)，也就是说在这个```TIME_WAIT```状态结束之前中，当前tcp元组是无法被复用的，除非设置了```SO_REUSEADDR```。

这里有两个关键参数，```SO_REUSEADDR```和```SO_REUSEPORT```，
首先字面上意思都是**复用**，具体概念如下：

参数 | 含义
---|---
SO_REUSEADDR | 允许连接ip地址在未完全断开的情况进行复用
SO_REUSEPORT | 在开启`SO_REUSEADDR`的前提下，允许连接端口地址进行复用

那么如果是开启复用并连接成功，在操作系统层面，假如多个文件句柄都绑定了系统的**ip+port**，系统会怎么处理呢，答案是负载均衡，即系统会根据请求进行分配，类似随机轮询的方式，对相同ip+port的连接进行交互。

这里可能有人会说，这样子不同客户端进程访问是否有权限越界问题呢，确实会有，所以基于安全考虑有一个约定：
> To prevent "port hijacking", there is one special limitation: All sockets that want to share the same address and port combination must belong to processes that share the same effective user ID

++所有要开启复用同一地址端口的连接必须属于同一个userID，而我们的上下文中是同一个进程或者说同一用户创建处理的，所以可以复用原来的连接，从而避免恶意劫持。++

### 程序示例
我们来看下程序如何实现

1. 传入复用连接的配置项
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

2. 检测当前监听的tcp元组是否正在监听
```go
func listener() (net.Listener, error) {
	lc := net.ListenConfig{
		Control: control,
	}
	if l, err := lc.Listen(context.TODO(), "tcp", ":8080"); err != nil {
	    // 端口未使用，返回err
		return nil, err
	} else {
		return l, nil
	}
}
```

3. 监听系统信号量
```go
func upgradeLoop(l *net.Listener, s *http.Server) {
	sig := make(chan os.Signal)
	signal.Notify(sig, syscall.SIGQUIT, syscall.SIGUSR2)
	for t := range sig {
		switch t {
		case syscall.SIGUSR2:
			// 接收升级信号量
			log.Println("Received SIGUSR2 upgrading binary")
			// Fork子进程优雅升级
			if err := spawnChild(); err != nil {
				log.Println(
					"Cannot perform binary upgrade, when starting process: ",
					err.Error(),
				)
				continue
			}
		case syscall.SIGQUIT:
		    // 接收杀掉当前进程的信号量
			s.Shutdown(context.Background())
			os.Exit(0)
		}
	}
}

// fork创建子进程，并在新进程之前更新覆盖全局父进程id
func spawnChild() error {
	// 获取当前启动传入可执行文件参数, 如./main
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

	// 存下当前进程, 这个id会在新进程启动之后kill掉
	ppid := os.Getpid()
	os.Setenv("APP_PPID", strconv.Itoa(ppid))

	// 启动新进程
	os.StartProcess(argv0, os.Args, &os.ProcAttr{
		Dir:   wd,
		Env:   os.Environ(),
		Files: files,
		Sys:   &syscall.SysProcAttr{},
	})

	return nil
}
```
4. 主协程的逻辑
```go
func main() {
	log.Println("Started HTTP API, PID: ", os.Getpid())
	var l net.Listener
    // 首次启动
    if fd, err := listener(); err != nil {
    	log.Println("Parent does not exists, starting a normal way")
    	l, err = net.Listen("tcp", ":8080")
    
    	if err != nil {
    		panic(err)
    	}
    } else {
    	// 新fork出来的，当前端口已被监听
    	l = fd
    	// 发送quit给父进程
    	killParent()
    	time.Sleep(time.Second)
    }
    
    // 启动server监听
    s := &http.Server{}
    	http.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
    		log.Printf("New request! From: %d, path: %s, method: %s: ", os.Getpid(),
    			r.URL, r.Method)
    	})
    go s.Serve(l)
    
    // 监听信号量
	upgradeLoop(&l, s)
}
```

#### 代码引用：
[zero-downtime-application项目源码](https://github.com/wutchzone/zero-downtime-application/blob/master/inherit-version/main.go)，其实核心在于新进程到旧进程的优雅迁移这个过程，只要理解了代码看起来就会清晰一点了。  

这也就是为什么main函数逻辑块需要兼容两个情形，一是正常server流程，一是接收旧进程的收尾。


### 拓展应用
关于上面的优雅重启触发机制是用户发送信号量```pkill -SIGUSR2```给进程，作为一个手动升级的无缝切换。  

其实基于这个功能可以进行拓展，比如监控服务加入连接探测，请求响应时间告警等，当达到某个触发机制，可以触发优雅重启，从而实现动态拉起的效果，当然后续还是需要复盘定位服务的问题在哪里，毕竟有时候重启并不能解决所有问题。

### 参考链接
- [Zero downtime API in Golang](https://wutch.medium.com/zero-downtime-api-in-golang-d5b6a52cc0ed)
- [Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)
- [基于K8S的优雅关闭](https://medium.com/over-engineering/graceful-shutdown-with-go-http-servers-and-kubernetes-rolling-updates-6697e7db17cf)
- [关于BSD套接字参数讨论](https://stackoverflow.com/questions/14388706/how-do-so-reuseaddr-and-so-reuseport-differ/14388707#14388707)