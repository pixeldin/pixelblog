---
title: "聊一聊Gin Web框架之前，看一眼httprouter"
date: 2020-06-16
categories:
- Go
- 网络编程
tags:
- Gin
showSocial: false
---

前言: Gin的词源是金酒, 又称琴酒, 是来自荷兰的一种烈性酒。
<!--more-->

在Go中，有一个经常提及的**web**框架，就是**gin web**，具备高性能，可灵活定制化的特点，既然它是如此被看好，在深入了解它之前，不妨先看下他是基于什么实现的。  


## 饮酒思源：httprouter
根据Git作者描述，**Gin**的高性能得益于一个叫**httprouter**的框架生成的，顺着源头看，我们先针对**httprouter**， 从**HTTP路由**开始，作为**Gin**框架的引入篇。

### route路由
在扎堆深入之前， 先梳理一下**路由**的概念：  
路由： 大概意思是通过转发数据包实现互联，比如生活中常见的物理路由器，是指内网与外网之间信息流的分配。  
同理，软件层面也有路由的概念，一般暴露在业务层之上，用于**转发**请求到合适的逻辑处理器。  
> The router matches incoming requests by the request method and the path.

程序的应用上，常见的如对外服务器把外部请求打到**Nginx网关**，再**路由**(转发)到内部服务或者内部服务的“控制层”，如Java的**springMVC**，Go的原生**router**等对不同请求转发到不同业务层。  
或者再具体化点说，比如不同参数调用同名方法，如**Java**的重载，也可以理解为程序根据参数的不同路由到相应不同的方法。

### httprouter
#### 功能现象：
**Git**的README文档上，**httprouter**开门见山的展示了它的一个常见功能，  
启动一个**HTTP**服务器，并且监听8080端口，对请求执行参数解析，仅仅几行代码，当我第一次见到这种实现时候，确实觉得**go**这种实现相当优雅。
```go
router.GET("/", Index)
//传入参数name
router.GET("/hello/:name", Hello)

func Hello(w http.ResponseWriter, r *http.Request) {
    //通过http.Request结构的上下文可以拿到请求url所带的参数
    params := httprouter.ParamsFromContext(r.Context())
    fmt.Fprintf(w, "hello, %s!\n", params.ByName("name"))
}

//启动监听
http.ListenAndServe(":8080", router)

```
----
### 接口实现
在观察了如何建立一个监听程序之后，挖掘这种优雅是如何封装实现之前，我们要先了解，在原生**Go**中，每个```Router```路由结构都实现了```http.Handler```接口，```Handler```只有一个方法体，就是```ServerHTTP```，它只有一个功能，就是处理请求，做出响应。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
题外话，**Go**中比较倾向于**KISS**或者单一职责，把每个接口的功能都单一化，有需要再进行组合，用组合代替继承，后续会把它当作一个编码规范来看。


#### net\http\server
```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
可以看到，在```Go```原生库中，```ServeHTTP()```实现体```HandlerFunc```就是个func函数类型，具体实现又是直接套用```HandlerFunc```进行处理，我没有在套娃哈，是不是有种“我实现我自己”的赶脚。  

All in all, 我们先抛开第三方库的封装，复习一下标准库，假如我们想用原生**http\server**包搭建一个HTTP处理逻辑，我们一般可以  
方式1：
  1. 定义一个参数列表是（ResponseWriter, *Request）的函数
  2. 将其注册到```http.Server```作为其```Handler```成员
  3. 调用**ListenAndServer**对**http.Server**进行监听

方式2：
  1. 定义一个结构，并且实现接口```ServeHTTP(w http.ResponseWriter, req *http.Request)```
  2. 将其注册到```http.Server```作为其```Handler```成员
  3. 调用**ListenAndServer**对**http.Server**进行监听

**示例如下：**
```go
//方式1
func SelfHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "<h1>Hello!</h1>")
}

//方式2
type HelloHandler struct {
}
//HelloHandler实现ServeHTTP()接口
func (* HelloHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	fmt.Fprint(w, "<h1>Hello!</h1>")
}

s := &http.Server{
	Addr:           ":8080",
	//方式1
	Handler:        SelfHandler,
	//方式2
	//Handler:      &HelloHandler{},
	ReadTimeout:    10 * time.Second,
	WriteTimeout:   10 * time.Second,
	MaxHeaderBytes: 1 << 20,
}
s.ListenAndServe()
```

**抛砖引玉：**  
以上就是**Go**标准库实现http服务的常用用法，现在进行拓展，假如我们需要通过url去获取参数，如**Get**请求，```localhost:8080/abc/1```
Q： 我们如何拿到abc或者1呢？  
A： 其实有个相对粗暴的方法，就是硬解：
- 利用```net/url```的```Parse()```函数把`8080`后面的那一段提取出来
- 使用```stings.split(ul, "/")```
- 利用下标进行参数范围

**示例如下**

```go
func TestStartHelloWithHttp(t *testing.T)  {
	//fmt.Println(path.Base(ul))

	ul := `https://localhost:8080/pixeldin/123`
	parse, e := url.Parse(ul)
	if e != nil {
		log.Fatalf("%v", e)
	}
	//fmt.Println(parse.Path)	// "/pixeldin/123"
	name := GetParamFromUrl(parse.Path, 1)
	id := GetParamFromUrl(parse.Path, 2)
	fmt.Println("name: " + name + ", id: " + id)	
}

//指定下标返回相对url的值
func GetParamFromUrl(base string, index int) (ps string) {
	kv := strings.Split(base, "/")
	assert(index < len(kv), errors.New("index out of range."))
	return kv[index]
}

func assert(ok bool, err error)  {
	if !ok {
		panic(err)
	}
}
```

**输出：**
```
name: pixeldin, id: 123
```
这种办法给人感觉相当暴力，而且需要记住每个参数的位置和对应的值，并且多个url不能统一管理起来，每次寻址都是遍历。尽管Go标准库也提供了一些通用函数，比如下面这个栗子：  
**GET**方式的url: ```https://localhost:8080/?key=hello```,  
可以通过```*http.Request```来获取，这种请求方式是在url中声明键值对，然后后台根据请求key进行提取。

```go
//摘取自：https://golangcode.com/get-a-url-parameter-from-a-request/
func handler(w http.ResponseWriter, r *http.Request) {

    keys, ok := r.URL.Query()["key"]
    
    if !ok || len(keys[0]) < 1 {
        log.Println("Url Param 'key' is missing")
        return
    }

    // Query()["key"] will return an array of items, 
    // we only want the single item.
    key := keys[0]

    log.Println("Url Param 'key' is: " + string(key))
}
```
但是，曾经沧海难为水。相信大家更喜欢开篇列举的那个例子，包括现在我们习惯的几个主流的框架，都倾向于利用url的位置去寻参，当然```httprouter```的优势肯定不止在这里，这里只是作为一个了解```httprouter```的切入点。
```go
router.GET("/hello/:name", Hello)
router.GET("/hello/*name", HelloWorld)
```
到这里先止住，后续我们来追踪它们封装之后的底层实现以及是如何规划url参数的。

---------------

### httprouter对ServerHTTP()的实现
前面提到，所有路由结构都实现了```http.Handler```接口的```ServeHTTP()```方法，我们来看下```httprouter```基于它的实现方式。  
#### julienschmidt\httprouter  
 在**httprouter**中，**ServeHTTP()** 的实现结构就叫```*Router```，它内部封装了用于检索url的tree结构，几个常用的布尔选项，还有几个也是基于```http.Handler```实现的默认处理器，它的实现如下：   

```go
// ServeHTTP makes the router implement the http.Handler interface.
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if r.PanicHandler != nil {
		defer r.recv(w, req)
	}

	path := req.URL.Path

	if root := r.trees[req.Method]; root != nil {
	    //getValue()返回处理方法与参数列表
		if handle, ps, tsr := root.getValue(path); handle != nil {
		    //匹配执行
			handle(w, req, ps)
			return
		} else if req.Method != http.MethodConnect && path != "/" {
			//...
		}
	}

	if req.Method == http.MethodOptions && r.HandleOPTIONS {
		// Handle OPTIONS requests
		//...
	} else if r.HandleMethodNotAllowed { // Handle 405
		//执行默认处理器...
	}

	// Handle 404
	if r.NotFound != nil {
		r.NotFound.ServeHTTP(w, req)
	} else {
		http.NotFound(w, req)
	}
}
```
到这里可以大致猜测它把处理method注入到内部trees结构，利用传入url在trees进行匹配查找，对执行链进行相应执行。
可以猜测这个```Router.trees```包含了handle和相应的参数，接着我们进入它的路由索引功能，来看下它是怎么实现++url匹配++与++参数解析++的。  

**结构梳理：**  
这个```trees```存在tree.go源文件中，其实是个map键值对，  
key是**HTTP methods**(如GET/HEAD/POST/PUT等)，method就是当前method与方法绑定上的节点  

我在源码补充些注释，相信大家容易看懂。
```go
// Handle registers a new request handle with the given path and method.
//
// For GET, POST, PUT, PATCH and DELETE requests the respective shortcut
// functions can be used.
// ...
func (r *Router) Handle(method, path string, handle Handle) {
	if len(path) < 1 || path[0] != '/' {
		panic("path must begin with '/' in path '" + path + "'")
	}

    //首次注册url，初始化trees
	if r.trees == nil {
		r.trees = make(map[string]*node)
	}

    //绑定http methods根节点，method可以是GET/POST/PUT等
	root := r.trees[method]
	if root == nil {
		root = new(node)
		r.trees[method] = root

		r.globalAllowed = r.allowed("*", "")
	}
    
    //对http methods方法树的路径划分
	root.addRoute(path, handle)
}
```

```Router.trees```map键值对的value是一个```node```结构体，每个**HTTP METHOD** 都是一个root节点，最主要的path分配是在这些节点的**addRoute()** 函数，  
简单理解的话, 最终那些前缀一致的路径会被绑定到这个树的同一个分支方向上，直接提高了索引的效率。  

下面我先列举出```node```几个比较重要的成员：

```go
type node struct {
	path      string
	//标识 path是否后续有':', 用于参数判断
	wildChild bool
	/* 当前节点的类型，默认是0，
	(root/param/catchAll)分别标识（根/有参数/全路径）*/
	nType     nodeType
	maxParams uint8
	//当前节点优先级, 挂在上面的子节点越多,优先级越高
	priority  uint32
	indices   string
	//满足前缀的子节点,可以延申
	children  []*node
	//与当前节点绑定的处理逻辑块
	handle    Handle
}
```

其中子节点越多，或者说绑定handle方法越多的根节点，**priority**优先级越高，作者有意识的对每次注册完成进行优先级排序。
引用作者的批注:
> This helps in two ways:
> - Nodes which are part of the most routing paths are evaluated first. This helps to make as much routes as possible to be reachable as fast as possible.
> - It is some sort of cost compensation. The longest reachable path (highest cost) can always be evaluated first. The following scheme visualizes the tree structure. Nodes are evaluated from top to bottom and from left to right.

优先级高的节点有利于handle的快速定位，相信比较好理解，现实中人流量密集处往往就是十字路口，类似交通枢纽。基于前缀树的匹配，让寻址从密集处开始，有助于提高效率。

**由浅入深：**  
我们先给路由器router注册几个作者提供的GET处理逻辑，然后开始调试，看下这个trees成员随着url新增有什么变化，
```go
router.Handle("GET", "/user/ab/", func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	//do nothing, just add path+handler
})

router.Handle("GET", "/user/abc/", func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	//do nothing, just add path+handler
})

router.Handle(http.MethodGet, "/user/query/:name", func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	routed = true
	want := httprouter.Params{httprouter.Param{"name", "gopher"}}
	if !reflect.DeepEqual(ps, want) {
		t.Fatalf("wrong wildcard values: want %v, got %v", want, ps)
	}
})
```

上述操作把【**RESTful的GET方法**，**url路径**，**匿名函数handler**】作为```router.Handler()```的参数，```router.Handler()```的操作上面我们已经简单的分析过了，主要节点划分在其中的```addRoute()```函数里面，下面我们简单过一下它的逻辑
```go
// 将当前url与处理逻辑放在当前节点
func (n *node) addRoute(path string, handle Handle) {
	fullPath := path
	n.priority++
	//提取当前url参数个数
	numParams := countParams(path)

	// 如果当前节点已经存在注册链路
	if len(n.path) > 0 || len(n.children) > 0 {
	walk:
		for {
			// 更新最大参数个数
			if numParams > n.maxParams {
				n.maxParams = numParams
			}

			// 判断待注册url是否与已有url有重合，提取重合的最长下标
			// This also implies that the common prefix contains no ':' or '*'
			// since the existing key can't contain those chars.
			i := 0
			max := min(len(path), len(n.path))
			for i < max && path[i] == n.path[i] {
				i++
			}

			/*
			    如果进来的url匹配长度大于于当前节点已有的url，则创建子节点
			    比如当前节点是/user/ab/ func1，
			    
			    新进来一个/user/abc/ func2，则需要创建/user/ab的子节点/ 和 c/
			    树状如下：
        			    |-/user/ab
        			    |--------|-/ func1  
        			    |--------|-c/ func2
        			    
			    之后如果再注册一个/user/a/ 与func3
			    则最终树会调整为：
        		优先级3 |-/user/a 
        		优先级2 |--------|-b 
        		优先级1 |----------|-/ func1
        		优先级1 |----------|-c/ func2
        		优先级1 |--------|-/ func3
			    
		    */
			if i < len(n.path) {
				child := node{
					path:      n.path[i:],
					wildChild: n.wildChild,
					nType:     static,
					indices:   n.indices,
					children:  n.children,
					handle:    n.handle,
					priority:  n.priority - 1,
				}

				// 遍历子节点，取最高优先级作为父节点优先级
				for i := range child.children {
					if child.children[i].maxParams > child.maxParams {
						child.maxParams = child.children[i].maxParams
					}
				}

				n.children = []*node{&child}
				// []byte for proper unicode char conversion, see #65
				n.indices = string([]byte{n.path[i]})
				n.path = path[:i]
				n.handle = nil
				n.wildChild = false
			}

			// Make new node a child of this node
			if i < len(path) {
				path = path[i:]

				if n.wildChild {
					n = n.children[0]
					n.priority++

					// Update maxParams of the child node
					if numParams > n.maxParams {
						n.maxParams = numParams
					}
					numParams--

					// Check if the wildcard matches
					if len(path) >= len(n.path) && n.path == path[:len(n.path)] &&
						// Adding a child to a catchAll is not possible
						n.nType != catchAll &&
						// Check for longer wildcard, e.g. :name and :names
						(len(n.path) >= len(path) || path[len(n.path)] == '/') {
						continue walk
					} else {
						// Wildcard conflict
						var pathSeg string
						if n.nType == catchAll {
							pathSeg = path
						} else {
							pathSeg = strings.SplitN(path, "/", 2)[0]
						}
						prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
						panic("'" + pathSeg +
							"' in new path '" + fullPath +
							"' conflicts with existing wildcard '" + n.path +
							"' in existing prefix '" + prefix +
							"'")
					}
				}

				c := path[0]

				// slash after param
				if n.nType == param && c == '/' && len(n.children) == 1 {
					n = n.children[0]
					n.priority++
					continue walk
				}

				// Check if a child with the next path byte exists
				for i := 0; i < len(n.indices); i++ {
					if c == n.indices[i] {
					    //增加当前节点的优先级，并且做出位置调整
						i = n.incrementChildPrio(i)
						n = n.children[i]
						continue walk
					}
				}

				// Otherwise insert it
				if c != ':' && c != '*' {
					// []byte for proper unicode char conversion, see #65
					n.indices += string([]byte{c})
					child := &node{
						maxParams: numParams,
					}
					n.children = append(n.children, child)
					n.incrementChildPrio(len(n.indices) - 1)
					n = child
				}
				n.insertChild(numParams, path, fullPath, handle)
				return

			} else if i == len(path) { // Make node a (in-path) leaf
				if n.handle != nil {
					panic("a handle is already registered for path '" + fullPath + "'")
				}
				n.handle = handle
			}
			return
		}
	} else { // Empty tree
		n.insertChild(numParams, path, fullPath, handle)
		n.nType = root
	}
}
```
以上的大致思路是，把各个handle func()注册到一棵url前缀树上面，根据url前缀相同的匹配度进行分支，以提高路由效率。

**参数查找：**  
接下来我们看下**httprouter**是怎么将参数**param**封装在**上下文**里面的：  
不难猜测分支划分的时候是通过判断关键字“:”来提取预接收的参数，这些参数是存字符串(字典)键值对，底层存储在一个```Param```结构体：

```go
type Param struct {
	Key   string
	Value string
}
```

关于上下文的概念在其他语言也挺常见，如**Java Spring**框架中的```application-context```，用来贯穿程序生命周期，用于管理一些全局属性。  
**Go**的上下文也在不同框架有多种实现，这里我们先初步了解Go程序最顶级的上下文是```background()```，是所有子上下文的来源，类似于Linux系统的init()进程。

先举个栗子，简单列举在**Go**的**context**传参的用法：

```go
func TestContext(t *testing.T) {
	// 获取顶级上下文
	ctx := context.Background()
	// 在上下文写入string值, 注意需要返回新的value上下文
	valueCtx := context.WithValue(ctx, "hello", "pixel")
	value := valueCtx.Value("hello")
	if value != nil {
	    /*
	        已知写入值是string，所以我们也可以直接进行类型断言
	        比如： p, _ := ctx.Value(ParamsKey).(Params)
	        这个下划线其实是go断言返回的bool值
	    */
		fmt.Printf("Params type: %v, value: %v.\n", reflect.TypeOf(value), value)
	}
}
```

**输出：**  

```shell
Params type: string, value: pixel.
```

在**httprouter**中，封装在```http.request```中的上下文其实是个**valueCtx**，类型和我们上面的栗子中**valueCtx**是一样的，框架中提供了一个从上下文获取**Params**键值对的方法，
```go
func ParamsFromContext(ctx context.Context) Params {
	p, _ := ctx.Value(ParamsKey).(Params)
	return p
}
```
利用返回的Params就可以根据key获取我们的目标值了，

```
params.ByName(key)
```

经过追寻，**Params**是来自一个叫```getValue(path string) (handle Handle, p Params, tsr bool)```的函数，还记得上面列举的*Router路由实现的```ServeHTTP()```接口吗？  


```go
//ServeHTTP() 函数的一部分
if root := r.trees[req.Method]; root != nil {
    /**   getValue()，返回处理方法与参数列表  **/
	if handle, ps, tsr := root.getValue(path); handle != nil {
	    //匹配执行， 这里的handle就是上面的匿名函数func
		handle(w, req, ps)
		return
	} else if req.Method != http.MethodConnect && path != "/" {
		//...
	}
}
```
```ServeHTTP()```其中有个```getValue```函数，它的返回值有两个重要成员：当前路由的处理逻辑和url参数列表，所以在路由注册的时候我们需要把params作为入参传进去。
像这样子：

```go
router.Handle(http.MethodGet, "/user/query/:name", 
//匿名函数func，这里的ps参数就是在ServeHttp()的时候帮你提取的
func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	fmt.Println(params.ByName("name"))
})
```

```getValue()```在对当前节点n进行类型判断，如果是‘param’类型(在addRoute的时候已经根据url进行分类)，则填充待返回参数Params。

```go
//-----------go
//...github.com/julienschmidt/httprouter@v1.3.0/tree.go:367
switch n.nType {
case param:
	// find param end (either '/' or path end)
	end := 0
	for end < len(path) && path[end] != '/' {
		end++
	}
	
    //遇到节点参数
	if p == nil {
		// lazy allocation
		p = make(Params, 0, n.maxParams)
	}
	i := len(p)
	p = p[:i+1] // expand slice within preallocated capacity
	p[i].Key = n.path[1:]
	p[i].Value = path[:end]
//...
```

**流程梳理：**  
So far，我们再次归纳一下```httprouter```的路由过程：
1. 初始化创建路由器router
2. 注册签名为：```type Handle func(http.ResponseWriter, *http.Request, Params)```的函数到该router
3. 调用HTTP通用接口```ServeHTTP()```，用于提取当前url预期的参数并且供业务层使用


----
以上这是这两天对httprouter的了解，顺便对**go**的Http有了进一步的认知，后续将尝试进入gin中，看下Gin是基于httprouter做了什么拓展以及熟悉常见用法。

### 参考链接：
**julienschmidt/httprouter**  
https://github.com/julienschmidt/httprouter