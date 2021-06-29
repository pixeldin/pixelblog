---
title: "聊一聊Go网络编程(二)--TCP连接管理"
date: 2021-05-16

categories:
- Go
- 网络编程
tags:
- TCP
- 连接池

showSocial: false
autoThumbnailImage: false
thumbnailImagePosition: "left"
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/net-2.png
---

上一篇我们聊到在Go中是如何发起一个TCP连接的，以及列举一个全双工的Demo，这次接着填补上一节留下的坑，连接池管理。
<!--more-->


### 前言
在上一节提到，计算机连接的每次tcp建立，都要进行三次握手，而为了避免频繁创建销毁，并且维持连接的活性，引入了keepalive的api，然而keepalive仅仅是告诉tcp链接在空闲时候进行探测（有别于HTTP），如果我们要真正复用连接的话，可以采用连接池，示例图如下。  
![image](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/tcp-pool.png)

### 连接池特性
如上图所示，一个连接池应该具备什么特性呢？  
1. 使用者可以从池子获取连接
2. 连接用完可以归还
3. 连接数有个**上限**，避免频繁创建导致连接复用率低
4. 对不健康的连接进行善后处理(关闭连接)，并把活跃连接数减一

### 边界情况
除了满足连接池的常用操作之外，我们要考虑如果连接数达到上限，而没有空闲连接怎么办，这里可以使用Go原生```sync```包```Mutex```自带的“**让步**”功能，类似于**Java**的```yield()```方法，释放当前互斥锁，并且等待其他连接归还时触发条件并**唤醒**。

#### 抛砖引玉
我们先用一个例子来实现等待与唤醒，这里主要是熟悉```wait()```和```Signal()```函数的用法。
- wait()  
当```wait()```执行时，会做两个操作：
    1. 释放当前绑定的互斥锁，源码执行了```c.L.Unlock()```，所以并不会阻塞长期占有资源，会释放让给其他协程唤醒。
    2. 把当前函数所在协程加入等待唤醒的队列
- Signal()  
    唤醒等待的协程有两个触发条件：
    1. 同一个互斥cond实例执行了```Signal()```函数
    2. ```wait()```函数之前等待的```for{}```条件被打破

**完整示例：**  
这里我们使用两个协程，id为<G-等待方>的协程先执行，并且在条件未允许处释放互斥锁，并等待唤醒，在函数```ForWhileSignal()```中实现。
```go
func ForWhileSignal(sig *signal, gid string, mutx *sync.Mutex, cd *sync.Cond) {
	mutx.Lock()
	defer func() {
		log.Print(gid + " 执行defer()\n")
		mutx.Unlock()
	}()

    // 等待打破条件
	for !sig.ready {
	    // 等待ready信号
		log.Print(gid + " 等待唤醒...\n")
		cd.Wait()
	}

	log.Print(gid + " 被叫醒! \n")
}
```
另外一方，id为<G-唤醒方>的协程后续执行唤醒，在```Condition()```函数实现
```go
func Condition(sig *signal, gid string, mutx *sync.Mutex, cd *sync.Cond) {
	mutx.Lock()
	defer func() {
		log.Print(gid + " 执行defer()\n")
		mutx.Unlock()
	}()

	log.Print(gid + " do something...\n")
	time.Sleep(3 * time.Second)
	sig.ready = true
	log.Print(gid + "旋转开关，唤醒等待方...\n")
	cd.Signal()
}
```
#### 主函数代码
```go
// 协程通信信号
type signal struct {
	// 用于wait()条件的检测
	ready bool
}
func main() {
    // 构建互斥共享变量
	mtx := new(sync.Mutex)
	cond := sync.NewCond(mtx)
	signal := new(signal)
	
	go ForWhileSignal(signal, "G-等待方", mtx, cond)
	time.Sleep(100)
	go Condition(signal, "G-唤醒方", mtx, cond)

	for {
		time.Sleep(1)
	}
}
```
#### 输出示例：
```
2021/05/15 23:25:54 G-等待方 等待唤醒...
2021/05/15 23:25:54 G-唤醒方 do something...
2021/05/15 23:25:57 G-唤醒方 旋转开关，唤醒等待方...
2021/05/15 23:25:57 G-唤醒方 执行defer()
2021/05/15 23:25:57 G-等待方 被叫醒!
2021/05/15 23:25:57 G-等待方 执行defer()
```
#### 程序时序图
完整的流程图如下，Wait()会把当前协程加入等待队列，等待由相同cond实例持有者进行唤醒。

![wait-signal](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/wait-signal.png)
---
### 实现
根据上面的原生API，可以利用唤醒机制着手连接池的实现。
#### 定义连接池
```go
type Pool struct {
	*Option
	idle    *list.List  // 空闲队列(双向)链表
	actives int         // 总连接数
	mtx     *sync.Mutex // 同步锁
	cond    *sync.Cond  // 用于阻塞/唤醒
}
```

#### 创建连接
```go
// NewPool 创建连接池,初始化连接队列,更新队列大小
func NewPool(opt *Option) (p *Pool, err error) {
	idle := list.New()
	var conn *Conn
	for i := 0; i < opt.size; i++ {
		conn, err = NewConn(opt)
		if err == nil {
			// 加入队列
			idle.PushBack(conn)
		}
		// whether close all idle conn when one of err occurs?
	}

    // mutex用于隔离临界区
	mutx := new(sync.Mutex)
	// cond用于唤醒/等待当前函数块
	cond := sync.NewCond(mutx)

	p = &Pool{
		opt,
		idle,
		idle.Len(),
		mutx,
		cond,
	}
	return
}
```

#### 获取连接
获取连接本质上是对空闲列表的分支判断，其中连接池共享成员mtx的```wait()```函数是关键点。  
结合前面的例子，当++连接池没有可用连接并且连接数上限已满时候++，即当for条件不成立时，程序会唤醒当前函数栈。    

这里可以简单理解为唤醒其他协程需要的前提是：
1. cond实例```wait()```前面的for循环条件，即连接池空闲数大于0，或者连接数小于上限的条件被打破
2. 在归还连接时候，也就是```put()```函数的cond实例显示调用了```Signal()```

**详细实现：**
```go
/*
	Get 获取连接:
	空闲列表是否有库存:
		- 没有
			- 连接数是否达到上限
				- 是, 解锁, 阻塞等待唤醒
				- 否, 创建连接
		- 从队头出队获取
*/
func (p *Pool) Get() (c *Conn, err error) {
	p.mtx.Lock()
	defer p.mtx.Unlock()
	// 如果当前活跃大于限制数量, 阻塞等待
	for p.idle.Len() == 0 && p.actives >= p.size {
		log.Print("idle size full, blocking...")
		// 让出使用权并且释放mutex锁，当for条件不成立会自动唤醒
		p.cond.Wait()
	}
	// 空闲列表如果有且连接, 则从空闲队头获取
	if p.idle.Len() > 0 {
		c = p.idle.Remove(p.idle.Front()).(*Conn)
	} else {
		// 创建连接
		c, err = NewConn(p.Option)
		if err == nil {
			p.actives++
		}
	}
	return
}
```
#### 归还连接
归还连接需要注意判断连接是否健康，如果异常则不能归还连接池，并更新活跃连接数。
```go
/*
  Put() 归还连接
  - 连接是否异常
	- 是, 关闭异常连接
	- 否, 归还连接至队尾
  - 更新占用连接数, 唤醒等待方
*/
func (p *Pool) Put(c *Conn, err error) {
	p.mtx.Lock()
	defer p.mtx.Unlock()

	if err != nil {
	    // 如果当前连接异常，执行关闭
		if c != nil {
			c.Close()
		}
	} else {
	    // 健康连接放入队尾
		p.idle.PushBack(c)
	}
	p.actives--
	// 唤醒等待队列
	p.cond.Signal()
}
```

#### 示例程序
至此，一个基本的连接池操作就实现了，下面我们开始测试。  

创建一个初始化5个连接的连接池，并启用10个协程并发进行连接交互。
```go
var opt = &Option{
	addr:        "127.0.0.1:3000",
	// 初始化5个连接
	size:        5,
	readTimeout: 30 * time.Second,
	dialTimeout: 5 * time.Second,
	keepAlive:   30 * time.Second,
}

func TestNewPool(t *testing.T) {
	pool, err := NewPool(opt)
	if err != nil {
		t.Fatal(err)
	}

	for i := 0; i < 10; i++ {
		go func(id int) {
			if err := SendInPool(pool, "Uid-"+strconv.Itoa(id)); err != nil {
				log.Print("Send in pool err: ", err)
			}
			// 注意闭包的id传入值
		}(i)
	}

    // just holding
	for {
		time.Sleep(1)
	}
}

func SendInPool(p *Pool, uid string) (err error) {
	var c *Conn
	if c, err = p.Get(); err != nil {
		return
	}
	defer p.Put(c, err)
	msg := &body.Message{Uid: uid, Val: "pixelpig!"}
	rec, err := c.Send(context.Background(), msg)
	if err != nil {
		log.Print(err)
	} else {
		log.Print(uid, ", Msg: ", <-rec)
	}
	return
}
```

**输出示例：**  
```go
=== RUN   TestNewPool
2021/05/17 01:22:59 idle size full, blocking...
2021/05/17 01:22:59 idle size full, blocking...
2021/05/17 01:22:59 idle size full, blocking...
2021/05/17 01:22:59 idle size full, blocking...
2021/05/17 01:22:59 idle size full, blocking...
2021/05/17 01:22:59 Uid-4, Msg: ts(ns): 1621185779673605400, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-0, Msg: ts(ns): 1621185779673605400, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-6, Msg: ts(ns): 1621185779673605400, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-9, Msg: ts(ns): 1621185779672609700, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-3, Msg: ts(ns): 1621185779673605400, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-1, Msg: ts(ns): 1621185779690594100, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-8, Msg: ts(ns): 1621185779690594100, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-5, Msg: ts(ns): 1621185779690594100, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-2, Msg: ts(ns): 1621185779690594100, server: hello, pixelpig!
2021/05/17 01:22:59 Uid-7, Msg: ts(ns): 1621185779690594100, server: hello, pixelpig!
```
可以看到程序输出符合预期，因为连接池大小只有5，所以并发10个连接大概率会有5个连接需要等待排队，因此如上输出有5个协程在队列等待(blocking)。

### 拓展
以上是维护一个TCP连接池的简单示例，还有比较常见的Redis/Kafka等数据库连接等，本质上都是TCP连接。  
虽然引入连接池增加了维护成本，需要注意临界区的读写冲突，连接池大小控制等，但是可以有效减少频繁的连接创建。  
此外，上述例子使用的是Go内置的双向队列维护多个连接，其实还有一种更加优雅的实现，就是利用Go的通道原生特性“阻塞与唤醒”，具体可参考《Go语言实战》连接池的代码。

这里缓冲的是任意Closer实例，通用性更高，部分代码如下：
#### 连接池结构
```go
type Pool struct {
	m        sync.Mutex
	//可以管理任意实现了Closer接口的资源类型
	resource chan io.Closer
	maxSize  int
	usedSize int
	// 构建连接的工厂函数，返回实现io.Closer的实例
	factory  func() (io.Closer, error)
	closed   bool
}
```
其中用于通信的通道类型是```io.Closer```，经常能在框架中看到一个用于检验接口实现类是否完整实现了接口的方式，
```go
// 如果Conn没有实现io.Closer接口，则编译器会提示：
// Cannot use 'new(Conn)' (type *Conn) as type io.Closer 
// Type does not implement 'io.Closer'

var _ io.Closer = new(Conn)
```
下面是相关代码：
#### 获取连接
```go
//get resource
func (p *Pool) Acquire() (io.Closer, error) {
	p.m.Lock()
	defer p.m.Unlock()

	select {
	// 有闲置连接则取出
	case r, ok := <-p.resource:
		log.Println("Acquire:", "Shard Resource")
		if !ok {
			//管道已经关闭
			return nil, ErrPoolClosed
		}
		return r, nil
	default:
		if p.usedSize < p.maxSize {
			p.usedSize++
			log.Printf("Acquire:" + "New Resource." +
				"resource present size/max: %d/%d\n", p.usedSize, p.maxSize)
			return p.factory()
		} else {
			//log.Printf("Acquire:" +
			//	"block for pool's dry, present size: %d/%d", p.usedSize, p.maxSize)
			return nil, nil
		}
	}
}
```

#### 释放并归还连接
```go
func (p *Pool) Release(r io.Closer) {
	p.m.Lock()
	defer p.m.Unlock()

	if p.closed {
		r.Close()
		return
	}

	p.usedSize--

	select {
	//资源放回队列
	case p.resource <- r:
		log.Println("Release:", "into queue")
	//队列满的情况关闭资源
	default:
		log.Println("Release:", "Closing")
		r.Close()
	}
}
```

#### 他山之石
不知读者是否能感觉到，使用Go通道自带的语言特性实现的阻塞与唤醒的通信模型，看起来更加简介且优雅。  
相信对Go的并发哲学：“++不用通过共享内存来通信，而是通过通信来共享内存++”的理解又有更深的体会了。