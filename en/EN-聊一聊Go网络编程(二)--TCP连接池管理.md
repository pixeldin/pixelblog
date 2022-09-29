---
title: "Talk about Go: Network programming--TCP Connection Management"
date: 2021-05-16

categories:
- Go
- Network
- TCP
tags:
- Talk about Go

showSocial: false
autoThumbnailImage: false

---

In the last article, we talked about how to initiate a TCP connection in Go, and listed a full-duplex demo. Today, we will talk about connection pool management further.
<!--more-->


### Foreword
As mentioned in the previous section, a three-way handshake is required for each tcp establishment of a computer connection. In order to avoid frequent creation and destruction and maintain the activity of the connection, the keepalive api is introduced. However, keepalive only tells the tcp link when it is idle. For detection (different from HTTP), if we want to really reuse connections, we can use connection pooling, as shown in the following example.
![tcp-pool](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/tcp-pool/EN-conn-pool.png)

### Features
What characteristics should a connection pool have?  
1. Consumers can get connections from the pool
2. The connection can be returned when used up
3. There is an upper limit on the number of connections to avoid frequent creation and low connection reuse rate
4. Remediate unhealthy connections (close connections) and decrement the number of active connections by one

### Edge Case
In addition to satisfying the common operations of connection pools, we need to consider what if the number of connections reaches the upper limit and there are no idle connections. Here, we can use the native ```sync``` package ```Mutex``` that comes with Go. **Yield**" function, similar to **Java**'s ```yield()``` method, releases the current mutex, and waits for the return of other connections to trigger the condition and **wake**.

#### Step By Step
Let's use an example to implement wait and wake up. Here we are mainly familiar with the usage of ```wait()``` and ```Signal()``` functions.
- wait()  
When ```wait()``` executes, it does two things:
    1. Release the currently bound mutex. The source code executes ```c.L.Unlock()```, so it will not block the long-term occupied resources, and will release it to other coroutines to wake up.
    2. Add the coroutine where the current function is located to the queue waiting to wake up
- Signal()  
    There are two trigger conditions to wake up a waiting coroutine:
    1. The same mutex cond instance executes the ```Signal()``` function
    2. The ```for{}``` condition waiting before the ```wait()``` function is broken

**Code example**  
Here we use two coroutines, the coroutine whose id is <G-waiting side> executes first, and releases the mutex when the condition is not allowed, and waits for wake-up, which is implemented in the function ```ForWhileSignal()```.
```go
func ForWhileSignal(sig *signal, gid string, mutx *sync.Mutex, cd *sync.Cond) {
	mutx.Lock()
	defer func() {
		log.Print(gid + " execute defer()\n")
		mutx.Unlock()
	}()

    // wail till condition break
	for !sig.ready {
		log.Print(gid + "wait for notify...\n")
		cd.Wait()
	}

	log.Print(gid + " notify! \n")
}
```
the other side, id as <G-notify> will wakes the waiting side up, which is implemented in the ```Condition()``` function
```go
func Condition(sig *signal, gid string, mutx *sync.Mutex, cd *sync.Cond) {
	mutx.Lock()
	defer func() {
		log.Print(gid + " defer()\n")
		mutx.Unlock()
	}()

	log.Print(gid + " do something...\n")
	time.Sleep(3 * time.Second)
	sig.ready = true
	log.Print(gid + "Rotary switch to wake up waiting side...\n")
	cd.Signal()
}
```
#### Main function
```go
type signal struct {
	// detection of wait() conditions
	ready bool
}
func main() {
	mtx := new(sync.Mutex)
	cond := sync.NewCond(mtx)
	signal := new(signal)
	
	go ForWhileSignal(signal, "G-waiting", mtx, cond)
	time.Sleep(100)
	go Condition(signal, "G-notify", mtx, cond)

	for {
		time.Sleep(1)
	}
}
```
#### Output example：
```
2021/05/15 23:25:54 G-waiting wait for notify......
2021/05/15 23:25:54 G-notify do something...
2021/05/15 23:25:57 G-notify Rotary switch to wake up waiting side...
2021/05/15 23:25:57 G-notify defer()
2021/05/15 23:25:57 G-waiting notify!
2021/05/15 23:25:57 G-waiting defer()
```
#### Timing diagram
The complete flow chart is as follows, wait() will add the current coroutine to the waiting queue, waiting for the same cond instance holder to wake up.

![wait-signal](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/wait-signal.png)

---

### Get's hands dirty
According to the above native API, the implementation of the connection pool can be started by using the wake-up mechanism.
#### Define
```go
type Pool struct {
	*Option
	idle    *list.List  // free queue (doubly) linked list
	actives int         // total connections
	mtx     *sync.Mutex 
	cond    *sync.Cond  // for blocking/awakening
}
```

#### Create a connection
```go
// initialize connection queue, update queue size
func NewPool(opt *Option) (p *Pool, err error) {
	idle := list.New()
	var conn *Conn
	for i := 0; i < opt.size; i++ {
		conn, err = NewConn(opt)
		if err == nil {
			// enqueue
			idle.PushBack(conn)
		}
		// whether close all idle conn when one of err occurs?
	}

    // mutex use for quarantine area
	mutx := new(sync.Mutex)
	// cond used to wake/wait the current function block
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

#### Get connected
Getting a connection is essentially a branch judgment on the free list, where the ```wait()``` function of the connection pool shared member mtx is the key point.  
Combined with the previous example, when the ++ connection pool has no available connections and the upper limit of the number of connections is full, that is, when the for condition does not hold, the program will wake up the current function stack.  

This can be simply understood as the premise required to wake up other coroutines:  
1. The for loop condition in front of the cond instance ```wait()```, that is, the condition that the idle number of the connection pool is greater than 0, or the condition that the number of connections is less than the upper limit is broken.
2. When returning the connection, that is, the cond instance of the ```put()``` function shows that ```Signal()``` is called.

**Implement detail:**
```go
/*
	Get() :
	Is the free list in idle queue:
		- no
			- Whether the number of connections has reached the upper limit
				- yes, unlocked, blocked waiting to wake up
				- no, Create a connection
		- pop from head of the queue
*/
func (p *Pool) Get() (c *Conn, err error) {
	p.mtx.Lock()
	defer p.mtx.Unlock()
	// If the current activity over the limit number, block and wait
	for p.idle.Len() == 0 && p.actives >= p.size {
		log.Print("idle size full, blocking...")
		// cancle possessing and release the mutex lock, it will wake up automatically when the for condition does not hold
		p.cond.Wait()
	}
	if p.idle.Len() > 0 {
		c = p.idle.Remove(p.idle.Front()).(*Conn)
	} else {
		c, err = NewConn(p.Option)
		if err == nil {
			p.actives++
		}
	}
	return
}
```
#### Return Connection
Returning the connection requires attention to determine whether the connection is healthy. If it is abnormal, the connection pool cannot be returned, and the number of active connections is updated.
```go
/*
  Put() 
  - Is the connection alive?
	- no, close it
	- yes, return link to end of line
  - Update the number of occupied connections, wake up the waiting side
*/
func (p *Pool) Put(c *Conn, err error) {
	p.mtx.Lock()
	defer p.mtx.Unlock()

	if err != nil {
		if c != nil {
			c.Close()
		}
	} else {
		p.idle.PushBack(c)
	}
	p.actives--
	p.cond.Signal()
}
```

#### Example

Let's create a connection pool that initializes 5 connections, and enables 10 coroutines to interact with connections concurrently.
```go
var opt = &Option{
	addr:        "127.0.0.1:3000",
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

**Output:**  
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
It can be seen that the program output is in line with expectations, because the connection pool size is only 5, so there is a high probability that 5 connections need to be queued for 10 concurrent connections, so there are 5 coroutines waiting in the queue for the above output (blocking).

The above is a simple implementation of a connection pool. If you are interested, you can know more with the project.  
**Link:** https://github.com/pixeldin/pool

### What's more
The above is a simple example of maintaining a TCP connection pool, as well as the more common database connections such as Redis/Kafka, which are essentially TCP connections.
Although the introduction of connection pool increases maintenance costs, you need to pay attention to read and write conflicts in critical sections, and control the size of connection pools, but it can effectively reduce frequent connection creation.  
In addition, the above example uses the built-in two-way queue of Go to maintain multiple connections. In fact, there is a more elegant implementation, which is to use the native channel feature of Go to "block and wake up". For details, please refer to the connection pool in "Go in action" code.  

The buffer here is any Closer instance, which is more versatile. Part of the code is as follows:
#### Define the Pool 
```go
type Pool struct {
	m        sync.Mutex
	// manage any resource type that implements the Closer interface
	resource chan io.Closer
	maxSize  int
	usedSize int
	// A factory function that builds a connection, returning an instance that implements io.Closer
	factory  func() (io.Closer, error)
	closed   bool
}
```
The channel type used for communication is ```io.Closer```. You can often see a way to check whether the interface implementation class fully implements the interface in the framework.
```go
// If Conn does not implement the io.Closer interface, 
// the compiler will prompt:
// Cannot use 'new(Conn)' (type *Conn) as type io.Closer 
// Type does not implement 'io.Closer'

var _ io.Closer = new(Conn)
```

#### Acquire
```go
//get resource
func (p *Pool) Acquire() (io.Closer, error) {
	p.m.Lock()
	defer p.m.Unlock()

	select {
	// take out if there is an idle connection
	case r, ok := <-p.resource:
		log.Println("Acquire:", "Shard Resource")
		if !ok {
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

#### Release and return the connection
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
	// put the resource back into the queue
	case p.resource <- r:
		log.Println("Release:", "into queue")
	// close resources when the queue is full
	default:
		log.Println("Release:", "Closing")
		r.Close()
	}
}
```

#### Stone of Other Mountains
I wonder if you can feel that the communication model of blocking and awakening implemented by using the language features that comes with the Go channel looks more concise and elegant.  
I have a deeper understanding of Go's concurrency philosophy: "*Do not communicate through shared memory, but share memory through communication*".