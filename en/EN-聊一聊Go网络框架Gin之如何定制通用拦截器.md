---
title: "Talk about Go: How to custom build a gateway interceptor in Gin framwork"
date: 2021-10-15
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/barrier.png
categories:
- Go
- microservice
tags:
- gin 
- interceptor

metaAlignment: center
---

Middleware often used as a level "security check", for core components such as request header/request body verification, caching, and current limiting.
<!--more-->
![security check](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/airport45.jpg)

## Foreword
Gateway middleware is an event stream processing execution channel. It is a request flow controls, often used as core components such as request header/request body verification, caching, and current limiting. Today, let's talk about gin framwork and build something.
## Module Combing
Let's first compare the **Go standard library** server and **Gin** framework startup process, and then start from ```Gin```.
### Go Server
We know that the Go standard library needs to go through the following steps to start the HTTP listener function:
- 流程概述  
Register Handler → ListenAndServer → for pool Accept → execute handler → call ```ServerHTTP()``` func internal  
```go
func main() {
	hf := func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello there, from %q", html.EscapeString(r.URL.Path))
	}

	http.HandleFunc("/bar", hf)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
Now let's go to the source code and see what it does:    
First, ```HandleFunc()```use the ```ServeMux```struct, there is a core inside which is ```muxEntry``` as a mapping between handler and url request path.
```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry // the Handler registration dictionary is used to return the corresponding handler function
	...
}
// the mapping of handler and url
type muxEntry struct {
	h       Handler
	pattern string
}
```
When the program starts, the main function above binds the ```hf``` and "/bar" paths relative to each other. 

Next, there is a simple implementation of multiplexing. We know that the cost of creating coroutines in Go is not high, and usually a request cycle does not last too long.
As you can see in the source code, each time Go accepts an HTTP connection, it is essentially a new Go coroutine for processing. The entire flowchart is as follows:
![HTTP request flow](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/http-req-flow.png)

### How Gin handle this under the water?

After observing the processing flow of the standard library, in fact, all the HTTP server implementation structures in Go are essentially by:
1. Using url to locate the registered logic function
2. Taking out the mapping ```handler()``` function to call

**Gin** framework is actually follow the main step. The optimization point is that the binding relationship between ```url``` and ```handler fun``` in **Gin** is on a prefix tree, which completely refers to ``` `HttpRouter```.  
If you are interested, you can refer to my previous blog post before:
[Before talking about the Gin web framework, let's take a look at httprouter](/en/2020/06/before-talking-about-the-gin-web-framework-take-a-look-at-httprouter/)

---- 
## Getting Started with Middleware by Gin
### Header Interrupted
We design a Header that must have a request entry with our desired value. If the required Header is lost, subsequent calls will not be made and the http request will be interrupted.  

**Code Example：**
```go
func init() {
	if err := os.Setenv("PORT", "8099"); err != nil {
		panic(err)
	}
}

var (
	H_KEY    = "h-key"  //business header key
)

// Declare a simple handler for pingpong as a request accepting behavior
func helloFunc(c *gin.Context) {
	const TAG = "PingPong"
	c.JSON(comdel.Success, comdel.SimpleResponse(200, TAG))
	return
}

func main() {
	r := gin.Default()
	// passing the header checker function as a barrier
	r.POST("/hello-with-header", middle.HeaderCheck(H_KEY), helloFunc)
	e := r.Run()
	fmt.Printf("Server stop with err: %v\n", e)
}
```

The routing relationship from ```/hello-with-header``` to ```middle.HeaderCheck(H_KEY)``` is registered above, which is equivalent to using the function ```HeaderCheck``` as ```helloFunc() ```'s upstream interceptor.  

Then we look at the specific code of the middleware:
```go
func HeaderCheck(key string) func(c *gin.Context) {
	return func(c *gin.Context) {
		kh := c.GetHeader(key)
		if kh == "" {			
			c.JSON(http.StatusOK, &comdel.Response{Code: comdel.Unknown, Msg: "lacking necessary header"})
			// illegal request, terminate the current process
			c.Abort()
			return
		}
		// normal request, and the execution chain is called down
		c.Next()
	}
}
```
**Output：**
You can see that ```Header``` is missing the key we expected, so it was intercepted by the middleware.
![gin-post-man-demo](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-lack-header.png)

So in theory, you can do any logic you want to verify in the middleware. Let's organize a body verification middleware. The function is:  
Accept any target type structure, check it against the target type when the HTTP request comes in, and use the ```ShouldBindBodyWith()``` of the gin framework to determine whether the current structure is legal.

### Structure verification (reflection, runtime detection)

**Pre-knowledge**：We all know that as a static language, Go's generics are not very mature, so if the program wants to express any type at runtime, it needs to use the reflection feature to judge the type of unknown parameters at runtime.
1. Use ```interface{}``` for compatible input types.  
2. If the incoming type is not nil, the parameter type is restored to the generic ```interface{}``
3. Recover the original structure and pass in the ```gin``` built-in verification function ```ShouldBindBodyWith()```  

Let's get out hands dirty, now write a runtime body structure detection, pass in the structure of the expected target type, and verify the fields
```go
// Construct return error format corresponding
func NewBindFailedResponse(tag string) *Response {
	return &Response{Code: WrongArgs, Msg: "wrong argument", Tag: tag}
}

// reqVal represents a specific struct type
func ReqCheck(reqVal interface{}) func(ctx *gin.Context) {
	var reqType reflect.Type = nil
	if reqVal != nil {
	    // restore from interface{}, extract the original value type
		value := reflect.Indirect(reflect.ValueOf(reqVal))
		// get the validation body primitive type at runtime
		reqType = value.Type()
	}

	return func(c *gin.Context) {
		tag := c.Request.RequestURI

		var req interface{} = nil
		if reqType != nil {
		    // primitive type
			req = reflect.New(reqType).Interface()
			// primitive type check
			if err := c.ShouldBindBodyWith(req, binding.JSON); err != nil {
				c.JSON(http.StatusOK, NewBindFailedResponse(tag))
				// bindBody err, terminate the execution chain
				c.Abort()
				return
			}
		}
		c.Next()
	}
}
```
**Mock struct:**
```go
// declare the request parameter structure, the id must be passed
type PingReq struct {
	Name string `json:"name"`
	// tag required
	Id   string `json:"id" binding:"required"`
}

/*
    Before routing/hello-with-req helloFunc() processing logic,
    Inject ReqCheck to detect request body interception module
*/
r.POST("/hello-with-req", middle.ReqCheck(bizmod.PingReq{}), helloFunc)
```
**Request example:**  
Legal parameters by convention:  
![Legal parameters](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-req-check-0.png)  

Incoming convention violation missing id parameter:  
![convention violation](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/go-gin/mid-req-check-1.png)
Both branches of the program are in line with our expectations, so the middleware of ```middle.ReqCheck(tar $Type)``` can be used to intercept requests in the future, so that the program can generally run on ```$Type``` Only when the type is checked according to the incoming ```$Type```, the repetitive code of the business function is reduced.  

The follow-up core business only needs to be handed over to ```helloFunc``` for execution.   

**It is equivalent to eliminating the hidden dangers of passengers entering and leaving the train station at the security checkpoint, and the passengers who get on the train must have passed the security check first.**

## Summary
Going back to the previous description, in theory, middleware is particularly suitable for accessing common modules, such as signature verification, gateway rete limiting, log reporting, cache, etc.  

This series will have the opportunity to carry out more specific implementation examples and analysis of business scenarios.