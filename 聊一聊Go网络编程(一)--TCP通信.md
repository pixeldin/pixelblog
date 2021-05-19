---
title: "聊一聊Go网络编程(一)--TCP通信"
date: 2021-05-15

categories:
- Go
- 后端
- 网络编程
tags:
- TCP
showSocial: false
---

在网络分层的七层协议中，我们知道TCP处于HTTP层的下方，本质上HTTP连接是基于底层的TCP连接建立的。 
<!--more-->


### TCP协议概要
在网络分层的七层协议中，我们知道TCP处于HTTP层的下方，本质上HTTP连接是基于底层的TCP连接建立的。  
**TCP连接标识**: 计算机之间在建立网络连接，也就是俗称的握手，本质上是两个文件句柄的关联，即fd，每个网络连接由四个属性唯一标识：<源IP,源端口,目标IP,目标端口>，因此一台机器的连接数受文件句柄```ulimit```的限制。   
**操作系统接口**: Socket套接字

#### 长连接KeepAlive的对比
- **HTTP keepalive**  
众所周知，HTTP连接是无状态的，通常连接用完就销毁，开启keepalive可以告知其保持连接一段时间，避免频繁连接重建。
- **TCP keepalive**  
    > Many existing TCP protocols support this way of error handling by defining some sort of heartbeat mechanism that requires each endpoint to send PING/PONG probes at a regular interval in order to detect both networking problems, as well as service health.
    
    TCP有别于HTTP，本身就是为了长连接而设定的，keepalive用于活性检测，可以理解为通过定义某种类型的心跳机制来支持这种错误处理方式，该心跳机制要求每个端点以规则的间隔发送ping/pong探测，以便检测网络问题以及服务健康。

### Linux网络参数
在Linux机器可以通过下列网络参数设置TCP保活机制：
```
  # cat /proc/sys/net/ipv4/tcp_keepalive_time
  7200

  # cat /proc/sys/net/ipv4/tcp_keepalive_intvl
  75

  # cat /proc/sys/net/ipv4/tcp_keepalive_probes
  9
```
**上述是默认设置，表示初始创建连接在两小时(7200秒)之后，每75秒重新发送一次。如果连续9次未收到ACK响应，则连接被标记为断开。**

### Go API介绍
在Go原生net包中，有下列函数可以干涉TCP连接的保活机制：
- ```func (c *TCPConn) SetKeepAlive(keepalive bool) error```  
    是否开启连接检测
- ```func (c *TCPConn) SetKeepAlivePeriod(d time.Duration) error```  
    连接检测间隔，如果不设置默认使用所在操作系统参数设置

----

### 用例Demo
下面我们先用一个TCP连接demo进行交互，之后我们再用连接池把TCP连接进行集中管理。

#### 传输结构
简单定义两个结构用于客户端与服务端交互，传输协议用**json**示范
```go
type Message struct {
	Uid string
	Val string
}

type Resp struct {
	Uid string
	Val string
	Ts string
}
```

#### server端
```golang
const TAG = "server: hello, "

func transfer(conn net.Conn) {
	defer func() {
		remoteAddr := conn.RemoteAddr().String()
		log.Print("discard remove add:", remoteAddr)
		conn.Close()
	}()

	// 设置10秒关闭连接
	//conn.SetDeadline(time.Now().Add(10 * time.Second))

	for {
		var msg body.Message

		if err := json.NewDecoder(conn).Decode(&msg); err != nil && err != io.EOF {
			log.Printf("Decode from client err: %v", err)
			// todo... 仿照redis协议写入err前缀符号`-`，通知client错误处理
			return
		}

		if msg.Uid != "" || msg.Val != "" {
			//conn.Write([]byte(msg.Val))
			var rsp body.Resp
			rsp.Uid = msg.Uid
			rsp.Val = TAG + msg.Val
			ser, _ := json.Marshal(msg)

			conn.Write(append(ser, '\n'))
		}
	}
}

func ListenAndServer() {
	log.Print("Start server...")
	// 启动监听本地tcp端口3000
	listen, err := net.Listen("tcp", "0.0.0.0:3000")
	if err != nil {
		log.Fatal("Listen failed. msg: ", err)
		return
	}
	for {
		conn, err := listen.Accept()
		if err != nil {
			log.Printf("accept failed, err: %v", err)
			continue
		}
		go transfer(conn)
	}
}
```

#### client端
定义一个Conn连接类型，用来包装原生tcp和其他额外属性，包括上下文，结果通道等。
```go
type IConn interface {
	Close() error
}

// Conn 对应每个连接
type Conn struct {
	addr    string              // 地址
	tcp     *net.TCPConn        // tcp连接实例, 可以是其他类型
	ctx     context.Context
	writer  *bufio.Writer
	cnlFun  context.CancelFunc // 用于通知ctx结束
	retChan *sync.Map          // 存放通道结果集合的map, 属于统一连接
	err     error
}

// 为Conn实现Close()函数签名 关闭连接, 关闭消息通道
func (c *Conn) Close() (err error) {
	// 执行善后
	if c.cnlFun != nil {
		c.cnlFun()
	}

	// 关闭tcp连接
	if c.tcp != nil {
		err = c.tcp.Close()
	}

	// 关闭消息通道
	if c.retChan != nil {
		c.retChan.Range(func(key, value interface{}) bool {
			// 根据具体业务断言转换通道类型
			if ch, ok := value.(chan string); ok {
				close(ch)
			}
			return true
		})
	}
	return
}
```

定义连接的配置项option结构
```go
type Option struct {
	addr        string
	size        int
	readTimeout time.Duration
	dialTimeout time.Duration
	keepAlive   time.Duration
}
```

紧接着创建连接代码如下：
```go
func NewConn(opt *Option) (c *Conn, err error) {
	// 初始化连接
	c = &Conn{
		addr:    opt.addr,
		retChan: new(sync.Map),
		//err: nil,
	}

	defer func() {
		if err != nil {
			if c != nil {
				c.Close()
			}
		}
	}()

	// 拨号
	var conn net.Conn
	if conn, err = net.DialTimeout("tcp", opt.addr, opt.dialTimeout); err != nil {
		return
	} else {
		c.tcp = conn.(*net.TCPConn)
	}

	c.writer = bufio.NewWriter(c.tcp)

	//if err = c.tcp.SetKeepAlive(true); err != nil {
	if err = c.tcp.SetKeepAlive(false); err != nil {
		return
	}
	if err = c.tcp.SetKeepAlivePeriod(opt.keepAlive); err != nil {
		return
	}
	if err = c.tcp.SetLinger(0); err != nil {
		return
	}

    // 创建上下文管理
	c.ctx, c.cnlFun = context.WithCancel(context.Background())

	// 异步接收结果到相应的结果集
	go receiveResp(c)

	return
}
```

#### 异步接收结果
来看下异步接收结果的代码，其中的```receiveResp()```函数，主要进行异步轮询，有几个作用：
- 感知上下文关闭，通常是连接的cancel()被执行
- 接收server端的数据并写入结果通道```retChan```，其类型是并发安全的```sync.Map```
- 监听server的错误，对异常情况关闭连接
```go
// receiveResp 接收tcp连接的数据
func receiveResp(c *Conn) {
	scanner := bufio.NewScanner(c.tcp)
	for {
		select {
		case <-c.ctx.Done():
			// c.cnlFun() 被执行了, 如连接池关闭
			return
		default:
			if scanner.Scan() {
				// 读取数据
				rsp := new(body.Resp)
				if err := json.Unmarshal(scanner.Bytes(), rsp); err != nil {
					return
				}
				// 响应id与请求id对应
				uid := rsp.Uid
				if load, ok := c.retChan.Load(uid); ok {
					c.retChan.Delete(uid)
					// 消息通道
					if ch, ok := load.(chan string); ok {
						ch <- rsp.Ts + ": " + rsp.Val
						// 在写入端关闭
						close(ch)
					}
				}
			} else {
				// 错误, 合并了EOF
				if scanner.Err() != nil {
					c.err = scanner.Err()
				} else {
					c.err = errors.New("scanner done")
				}
				c.Close()
				return
			}
		}
	}
}
```

#### 发送请求
```go
/*
	Send 发送请求, 返回具体业务通道
	注意如果入参的msg消息体是interface{}类型, 最好根据业务进行
	类型断言校验, 避免server端解析出错，返回err值用于后续判断
	是否归还连接池。
*/
func (c *Conn) Send(ctx context.Context, msg *body.Message) (ch chan string, err error) {
	ch = make(chan string)
	c.retChan.Store(msg.Uid, ch)
	// 请求
	js, _ := json.Marshal(msg)

	_, err = c.writer.Write(js)
	if err != nil {
		return
	}

	err = c.writer.Flush()
	// 连接不关闭, 后续可以放入连接池
	//c.tcp.CloseWrite()
	return
}
```

#### 实例：
1. 启动server端监听:
```
=== RUN   TestListenAndServer
2021/05/10 16:58:20 Start server...
```
2. 发起请求：
```go
var OPT = &Option{
	addr:        "0.0.0.0:3000",
	size:        3,
	readTimeout: 3 * time.Second,
	dialTimeout: 3 * time.Second,
	keepAlive:   1 * time.Second,
}

func createConn(opt *Option) *Conn {
	c, err := NewConn(opt)
	if err != nil {
		panic(err)
	}
	return c
}

func TestSendMsg(t *testing.T) {
	c := createConn(OPT)
	msg := &body.Message{Uid: "pixel-1", Val: "pixelpig!"}
	rec, err := c.Send(context.Background(), msg)
	if err != nil {
		t.Error(err)
	} else {
		t.Logf("rec1: %+v", <-rec)
	}

	msg.Val = "another pig!"
	rec2, err := c.Send(context.Background(), msg)
	if err != nil {
		t.Error(err)
	} else {
		t.Logf("rec2: %+v", <-rec2)
	}
	t.Log("finished")
}
```
3. 客户端输出如下：
```
=== RUN   TestSendMsg
    TestSendMsg: conn_test.go:56: rec1: : pixelpig!
    TestSendMsg: conn_test.go:64: rec2: : another pig!
    TestSendMsg: conn_test.go:66: finished
--- PASS: TestSendMsg (9.94s)
PASS
```

----
### 超时与池化管理
上面是一个比较简单的点对点交互，后续其实还可以考虑连接交互超时的情况：
1. 虽然连接结果是异步响应，但是我们有必要对响应进行超时判断，防止单个连接持续阻塞
2. 我们要考虑复用，即把健康的连接放入连接池进行管理。

#### 超时判断
超时判断业界有许多做法，比较常见的是用一个```select{}```块与```time.After()```即可。  
下面我们来看下常见的实现：
```go
rec3, err := c.Send(context.Background(), msg)
if err == nil {
	select {
	case resp := <-rec3:
		t.Logf("rec3: %+v", resp)
		return
	case <-time.After(time.Second * 1):
		t.Error("Wait for resp timeout!")
		return
	}
} else {
	t.Error(err)
}
```
超时输出如下：
```
=== RUN   TestSendMsg
    TestSendMsg: conn_test.go:56: rec1: : pixelpig!
    TestSendMsg: conn_test.go:76: Wait for resp timeout!
--- FAIL: TestSendMsg (17.99s)
FAIL
```

#### 连接池管理
这里要考虑的情况稍微复杂点，可以先把难点列出来再逐个击破：
1. 池子的连接数上限
2. 空闲连接数更新
3. 连接获取与归还
4. 连接关闭

关于池化操作篇幅可能较长，详解在本系列的下一篇《聊一聊Go网络编程--TCP连接管理(二)》叙述。

### 参考链接
**Notes on TCP keepalive in Go**  
https://thenotexpert.com/golang-tcp-keepalive/  
**Using TCP keepalive under Linux**  
https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html