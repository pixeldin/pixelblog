---
title: "聊一聊Go网络框架Gin之如何定制通用拦截器"
date: 2021-08-27
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/barrier.png
categories:
- Go
- gin
tags:
- 网关中间件
- 拦截器

metaAlignment: center
---

middleware，常常用于作为请求头/请求体校验、缓存、限流等核心组件的关卡"安检"。
<!--more-->
![安检](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/airport45.jpg)

## 前言
网关中间件是事件流处理执行通道，它是请求流转控件，又叫middleware，常常用于作为请求头/请求体校验、缓存、限流等核心组件的关卡"安检"。
## 模块梳理
抛砖引玉，我们先来比对**Go标准库**server与**Gin**框架启动流程，再从```gin```开始切入。
### Go标准库Server
我们知道，Go标准库启动HTTP监听函数需要经过下面几个步骤：
- 流程概述  
注册Handler → ListenAndServer → for轮询Accept → 执行处理 → 调用内部实现体的ServerHTTP()函数  
```go
func main() {
	hf := func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello there, from %q", html.EscapeString(r.URL.Path))
	}

	http.HandleFunc("/bar", hf)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
下面我们进入源码，看下其中做了什么：  
首先，```HandleFunc()```底层封装了一个叫做```ServeMux```的结构体，内部有一个核心是```muxEntry```作为handler与url请求路径的映射。
```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry // Handler注册字典用于返回对应的处理函数
	...
}
// handler与定位url的映射
type muxEntry struct {
	h       Handler
	pattern string
}
```
当程序启动，上面的main函数相对于把hf和"/bar"这个路径绑在一起。  

接着，就是多路复用的简单实现，我们知道在Go里创建协程的成本其实不高，而且通常一个请求周期不会持续太久。  
源码中可以看到，Go每次接受一个HTTP连接本质上就是new一个Go协程进行处理，整个流程图如下：
![HTTP request flow](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/http-req-flow.png)

### Gin做了什么封装

OK，知道了主要流程之后，其实Go里面所有的HTTP server实现结构，本质上就是
1. 利用url定位到已注册的逻辑函数
2. 取出```handler()```函数进行调用。  

**Gin**框架其实也是这个主要过程，优化点在于在**Gin**里面```url```和```handler fun```的绑定关系是在一颗前缀树上面，完全参照了```HttpRouter```。  
有兴趣可以参考我在掘金之前的一篇博客梳理：
[聊一聊Gin Web框架之前，看一眼httprouter](https://juejin.cn/post/6844904115131121677)

---- 
## 中间件入门----gin常见案例
后续我们先来实现几个比较简单的中间件模板：
### Header拦截器
我们设计一个Header必须要有我们期望值的请求入口，如果丢失所需Header则不进行后续调用，中断http请求。  

**用法模板：**
```go
func init() {
	// 设置gin启动默认端口
	if err := os.Setenv("PORT", "8099"); err != nil {
		panic(err)
	}
}

var (
	H_KEY    = "h-key"  //业务header key
)

// 声明一个pingpong的简单handler，作为请求接受行为
func helloFunc(c *gin.Context) {
	const TAG = "PingPong"
	c.JSON(comdel.Success, comdel.SimpleResponse(200, TAG))
	return
}

func main() {
    // 创建gin实例
	r := gin.Default()
	// 校验header
	r.POST("/hello-with-header", middle.HeaderCheck(H_KEY), helloFunc)
	e := r.Run()
	fmt.Printf("Server stop with err: %v\n", e)
}
```

上面注册了```/hello-with-header```到```middle.HeaderCheck(H_KEY)```的路由关系，相当于把函数```HeaderCheck```作为```helloFunc()```的上游拦截器。  

接着看下中间件的具体代码：
```go
func HeaderCheck(key string) func(c *gin.Context) {
	return func(c *gin.Context) {
		// 获取header的值
		kh := c.GetHeader(key)
		if kh == "" {
			// header缺失
			c.JSON(http.StatusOK, &comdel.Response{Code: comdel.Unknown, Msg: "lacking necessary header"})
			// 请求不合法，终止当前流程
			c.Abort()
			return
		}
		// 请求正常，执行链往下调用
		c.Next()
	}
}
```
**演示结果：**
可以看到```Header```缺少我们预期要的key，所以被中间件拦截了。
![header拦截](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-lack-header.png)

所以理论上你可以在中间件做任何你想要校验的逻辑，我们再来整理一个body校验中间件，功能是：  
接受任意目标类型的结构体，在HTTP请求进来时候对其与目标类型进行校验，利用gin框架的```ShouldBindBodyWith()```来定位当前结构体是否合法。

### 结构体校验(反射、运行时检测)

**前置知识**：我们都知道目前作为静态语言，Go的泛型还不是很成熟，所以程序在运行时如果要表示传入任意类型，需要利用反射特性，对运行时未知参数进行类型判断
1. 用```interface{}```来进行兼容入参类型。  
2. 如果传入类型非nil，则对通用```interface{}```进行参数类型还原
3. 还原为原结构传入```gin```内置校验函数```ShouldBindBodyWith()```  

现在编写一个运行时body结构体检测，传入预期目标类型的结构，并对字段进行校验
```go
// 构造返回错误格式相应
func NewBindFailedResponse(tag string) *Response {
	return &Response{Code: WrongArgs, Msg: "wrong argument", Tag: tag}
}

// reqVal表示具体struct类型
func ReqCheck(reqVal interface{}) func(ctx *gin.Context) {
	var reqType reflect.Type = nil
	if reqVal != nil {
	    // 从interface{}还原，提取原值类型
		value := reflect.Indirect(reflect.ValueOf(reqVal))
		// 运行时: 拿到校验体原始类型
		reqType = value.Type()
	}

	return func(c *gin.Context) {
		tag := c.Request.RequestURI

		var req interface{} = nil
		if reqType != nil {
		    // 原始类型
			req = reflect.New(reqType).Interface()
			// 原始类型校验
			if err := c.ShouldBindBodyWith(req, binding.JSON); err != nil {
				// 结构体绑定出错
				c.JSON(http.StatusOK, NewBindFailedResponse(tag))
				// 终止执行链
				c.Abort()
				return
			}
		}
		// 无需校验, 执行链往下
		c.Next()
	}
}
```
**程序示例：**
```go
// 声明请求参数结构，id必传
type PingReq struct {
	Name string `json:"name"`
	// tag required表示id字段必传
	Id   string `json:"id" binding:"required"`
}

/*
    在路由/hello-with-req helloFunc()处理逻辑之前，
    注入ReqCheck检测请求体拦截模块
*/
r.POST("/hello-with-req", middle.ReqCheck(bizmod.PingReq{}), helloFunc)
```
**请求示例：**  
传入约定合法参数：  
![请求合法](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-req-check-0.png)  

传入约定违规缺失id参数：  
![请求缺失必选字段](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-req-check-1.png)
程序两个分支都符合我们的预期，所以，后续可以```middle.ReqCheck(tar $Type)```这个中间件来拦截请求，让程序通用地对```$Type```在运行时才根据传入```$Type```进行类型校验，减轻业务函数的重复代码。  

后续核心业务只需要交给```helloFunc```去执行就好了，**相当于把进出火车站的乘客隐患在安检处统一处理排除，上了火车的一定是安检通过的乘客。**

## 总结
回到前面的描述，理论上中间件特别适合做通用模块的接入，如签名校验/网关限流/日志上报/缓存等等。  

这个系列后续有机会将会展开更具体的业务场景的实现举例与分析。