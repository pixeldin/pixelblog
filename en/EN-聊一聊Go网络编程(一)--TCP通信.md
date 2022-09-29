---
title: "Talk about Go: Network programming--TCP Communication"
date: 2021-05-15

categories:
- Go
- Network
- TCP
tags:
- Talk about Go

showSocial: false
autoThumbnailImage: false

---

In the seven-layer protocol of network layering, we know that TCP is below the HTTP layer. In essence, the HTTP packet body parsing is established based on the underlying TCP connection.
<!--more-->


## TCP Protocol Summary  
**TCP Connection Distinct**: The establishment of a network connection between computers, also known as handshake, is essentially the association of two file handles, namely fd, each network connection is uniquely identified by four attributes: `<source IP, source port, destination IP, destination port >`, so the number of connections to a machine is limited by the file handle ```ulimit```.   
**OS Socket**: Operating system sockets serve as endpoints for establishing a two-way network communication link between server-side and client-side programs.

### Explanation of KeepAlive in different scenarios
- **HTTP keepalive**  
As we all know, HTTP connections are stateless. Usually, the connection is destroyed when it is used up. Turning on keepalive can tell it to keep the connection for a period of time and avoid frequent connection reconstruction.
- **TCP keepalive**  
    > Many existing TCP protocols support this way of error handling by defining some sort of heartbeat mechanism that requires each endpoint to send PING/PONG probes at a regular interval in order to detect both networking problems, as well as service health.

### Linux Network parameters
On Linux machines, the TCP keepalive mechanism can be set via the following network parameters:
```
  # cat /proc/sys/net/ipv4/tcp_keepalive_time
  7200

  # cat /proc/sys/net/ipv4/tcp_keepalive_intvl
  75

  # cat /proc/sys/net/ipv4/tcp_keepalive_probes
  9
```
The above is the default setting, which means that **the initial connection creation will be resent every 75 seconds after two hours (7200 seconds). If no ACK response is received 9 times in a row, the connection is marked as broken.**

### Code Demo
In the Go native net package, the following functions can set with the keep-alive mechanism of TCP connections:  
- ```func (c *TCPConn) SetKeepAlive(keepalive bool) error```  
    whether to enable connection detection
- ```func (c *TCPConn) SetKeepAlivePeriod(d time.Duration) error```  
    connection detection interval, by default will use the operating system parameter settings

----

Next, we first use a TCP connection demo to interact, and then we use the connection pool to centrally manage the TCP connection at next time.

**Transmission Structure**  
We define two structures for the interaction between the client and the server, and the transmission protocol is demonstrated by **json**
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

**Server side**  
```golang
const TAG = "server: hello, "

func transfer(conn net.Conn) {
	defer func() {
		remoteAddr := conn.RemoteAddr().String()
		log.Print("discard remove add:", remoteAddr)
		conn.Close()
	}()

	// set 10 seconds to close the connection
	//conn.SetDeadline(time.Now().Add(10 * time.Second))

	for {
		var msg body.Message

		if err := json.NewDecoder(conn).Decode(&msg); err != nil && err != io.EOF {
			log.Printf("Decode from client err: %v", err)
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
	// start listening on local tcp port 3000
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

**Client Side**  
Defines a Conn connection type that wraps native tcp and other extra properties, including context, result channel, etc.
```go
type IConn interface {
	Close() error
}

// for each connection
type Conn struct {
	addr    string              
	tcp     *net.TCPConn        // tcp connection can be any types implement of database(Redis,MySQL,Kafka)
	ctx     context.Context
	writer  *bufio.Writer
	cnlFun  context.CancelFunc // tsed to notify the end of ctx
	retChan *sync.Map          // the map that stores the channel result set, which belongs to the unified connection
	err     error
}

// Implement the Close() function signature for Conn to close the connection, close the message channel
func (c *Conn) Close() (err error) {
	if c.cnlFun != nil {
		c.cnlFun()
	}
	
	if c.tcp != nil {
		err = c.tcp.Close()
	}
	if c.retChan != nil {
		c.retChan.Range(func(key, value interface{}) bool {
			// convert the channel type according to the specific business assertion
			if ch, ok := value.(chan string); ok {
				close(ch)
			}
			return true
		})
	}
	return
}
```

**Defines the option of the connection**
```go
type Option struct {
	addr        string
	size        int
	readTimeout time.Duration
	dialTimeout time.Duration
	keepAlive   time.Duration
}
```

Then create the connection code as follows:
```go
func NewConn(opt *Option) (c *Conn, err error) {
	// initialize connection
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

	// dial
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

    // create context management
	c.ctx, c.cnlFun = context.WithCancel(context.Background())

	// receive results asynchronously to the corresponding result set
	go receiveResp(c)

	return
}
```

**Receive results asynchronously**  
The ```receiveResp()``` function, which mainly performs asynchronous polling, has several functions:
- Aware of context closure, usually the connection's cancel() is executed
- Receive data from the server and write to the result channel ```retChan```, whose type is concurrency-safe ```sync.Map```
- Listen for server errors and close connections for exceptions
```go
// receive the data from tcp connection
func receiveResp(c *Conn) {
	scanner := bufio.NewScanner(c.tcp)
	for {
		select {
		case <-c.ctx.Done():
			// c.cnlFun() is executed, if the connection pool is closed
			return
		default:
			if scanner.Scan() {
				rsp := new(body.Resp)
				if err := json.Unmarshal(scanner.Bytes(), rsp); err != nil {
					return
				}
				// the response id corresponds to the request id
				uid := rsp.Uid
				if load, ok := c.retChan.Load(uid); ok {
					c.retChan.Delete(uid)
					// message channel
					if ch, ok := load.(chan string); ok {
						ch <- rsp.Ts + ": " + rsp.Val
						// close on write side
						close(ch)
					}
				}
			} else {
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

**Sending request**  
```go
func (c *Conn) Send(ctx context.Context, msg *body.Message) (ch chan string, err error) {
	ch = make(chan string)
	c.retChan.Store(msg.Uid, ch)
	js, _ := json.Marshal(msg)

	_, err = c.writer.Write(js)
	if err != nil {
		return
	}

	err = c.writer.Flush()
	// the connection is not closed, could be put into the connection pool later
	//c.tcp.CloseWrite()
	return
}
```

### Running Steps
1. Start server-side listening:

```
=== RUN   TestListenAndServer
2021/05/10 16:58:20 Start server...
```
2. Make a request：

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

3. Client output：  

```
=== RUN   TestSendMsg
    TestSendMsg: conn_test.go:56: rec1: : pixelpig!
    TestSendMsg: conn_test.go:64: rec2: : another pig!
    TestSendMsg: conn_test.go:66: finished
--- PASS: TestSendMsg (9.94s)
PASS
```

----

### Timeout and Pooling
The above is a relatively simple point-to-point interaction. In fact, the connection interaction timeout can also be considered later:
1. Although the connection result is an asynchronous response, it is necessary for us to time out the response to prevent a single connection from continuing to block.
2. We need to consider reuse, that is, put healthy connections into the connection pool for management.

**Timeout judgment**  
There are many ways of judging timeout, the more common one is to use a ```select{}``` block and ```time.After()```.
Let's take a look at the common implementations:
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

**Example：**

```
=== RUN   TestSendMsg
    TestSendMsg: conn_test.go:56: rec1: : pixelpig!
    TestSendMsg: conn_test.go:76: Wait for resp timeout!
--- FAIL: TestSendMsg (17.99s)
FAIL
```

**Pool Management**  
The situation to be considered here is a little more complicated. You can list the difficulties first and then break them one by one:
1. The maximum number of connections in the pool
2. Update the number of idle connections
3. Connection acquisition and return
4. Connection closed

The length of the pooling operation may be long, and the detailed explanation is described in the next part of this series: [Talk about Go: TCP Connection Pool Management](/en/2021/05/talk-about-go-network-programming-tcp-connection-management/)

### Reference Link
**Notes on TCP keepalive in Go**  
https://thenotexpert.com/golang-tcp-keepalive/  
**Using TCP keepalive under Linux**  
https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html