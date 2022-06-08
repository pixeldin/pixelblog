---
title: "聊一聊Go语言的error处理"
date: 2020-03-15

categories:
- Go
tags:
- errors
showSocial: false
---

在Go语言中，错误处理是一个常见的操作
<!--more-->
## 前言
Go语言的错误处理是一个常见的操作，经常可以见到一个函数返回错误类型(error)，后续通过```if err != nil```来判断错误以及错误类型。  
这一次尝试通过**Go**内置的**error**接口，我们聊一聊Go语言的错误处理以及**Error**的惯例用法。


## 单一职责：Error接口
### 接口签名
```go
type error interface {
    Error() string
}
```
我们先看**Go**的src/builtin内置**error**接口，它只有一个**Error**()方法，返回一个**string**，用来备注错误信息。任何实现了这个方法的结构体都实现了**error**接口。

### 实现类
随便列举一个Go自带的栗子，实现了**Error接口**的结构体，比如```Go1.12/src/net/net.go```的```AddrError```
```go
type AddrError struct {
    Err  string
    Addr string
}

//使用指针接收器实现该Error()接口
func (e *AddrError) Error() string {
    if e == nil {
    	return "<nil>"
    }
    s := e.Err
    if e.Addr != "" {
    	s = "address " + e.Addr + ": " + s
    }
    return s
}
```


### 那么问题来了

#### 为什么接口实现用指针接收器的场景？  
在Go里面，使用指针实现接口有两个主要用途：  
1. 为了在实现该函数处可以修改指针调用者
2. 大结构使用指针可以减小拷贝，另外可以保证共享，维持全局一个类型，类似于单例。

#### 用指针接收器实现Error()方法
我们可以看到上面```AddrError```使用指针接收器**AddrError**实现 **Error()** 接口，结合上一个问题的用途分析，**Error()** 方法主要是为了第二点，唯一标识错误类型。  

在Go中，Error是一个可比较的接口，我们都知道，指针的比较是比较地址，如果通过结构体(值)比较，无法确定当前Error是自定义Error或者内置Error，如io.EOF。  

通过这种方式，我们可以在```err == io.EOF```等于true的时候，大胆地be sure这err不会是其他自定义Error实现类，一定来自于**io包**的内置**error**，这种内置错误更多作为一个全局变量贯穿在**Go**程序中，类似于单例。

其次，在自定义错误中，通过指针的```.(type)```断言，可以针对不同**error**类型进行判断，执行多态处理，统一使用指针进行实现方便断言判断处进行归纳。可以在src/encoding/json/decode.go找到几个常用错误类型。  

#### 程序示例:
下面是一个通过断言switch作出不同处理的例子
```go
var u user
err := json.Unmarshal([]byte({&quot;name&quot;:&quot;bill&quot;}), u)
switch e := err.(type) {
case *json.UnmarshalTypeError:
    log.Printf("UnmarshalTypeError: Value[%s] Type[%v]\n", e.Value, e.Type)
case *json.InvalidUnmarshalError:
    log.Printf("InvalidUnmarshalError: Type[%v]\n", e.Type)
default:
    log.Println(err)
}
```
如果是通过值实现```Error()```方法，在case判断处，需要归纳```*json.UnmarshalTypeError```以及```json.UnmarshalTypeError```，因为**通过值实现的函数调用方可以是指针或者是值。**

## 个性化：自定义Error
### 应用场景
当**Error**需要包装额外的信息，内置error又没有拓展空间时候，我偏要勉强怎么办？比如调用对象，程序栈信息等。  
曲线救国，可以使用自定义错误，并且把调用栈信息填入该**error**，下面通过列举一个**自定义Error**的demo，尝试在**自定义Error**中加入上下文信息。

上下文信息指的是对当前结构进行一些属性关联（如当前结构体类型，时间，情景要素等），可以封装一个**context**属性或者新增几个所需属性，这里列举几个简单自定义错误类型，添加不同场景的上下文数据以及调用栈。

### 程序栗子：
新建三个自定义错误，分别适应不同的场景：类型错误/容量错误/时间错误，并且在**TypeError**嵌入我们需要的：
- **上下文**，这里简单用string作描述
- **栈信息**，可能用于追踪程序的执行
- **Type**字段，这里用于后续本例调试
```go
package main

import (
    "fmt"
    "reflect"
)

//自定义错误1
type TypeError struct {
    //上下文
    context string
    Type reflect.Type
    trace string
}

//具体Error()实现，返回类型，上下文，以及调用栈描述
func (tye* TypeError) Error() string {
    return fmt.Sprintf("Type of TypeError %s, contex: %s, trace[%s]",
        tye.Type, tye.context, tye.trace)
}

//自定义错误2
type SizeError struct {
    context string
    Type reflect.Type
}

func (sie * SizeError) Error() string {
    return fmt.Sprintf("SizeError context: %s", sie.context)
}

//自定义错误3
type UserError struct {
    context string
    Type reflect.Type
}

func (tie *UserError) Error() string {
    return fmt.Sprintf("UserError %s, contex: %s", tie.context)
}

```
**执行栈的获取示例：**

```go
/*
	https://www.komu.engineer/blogs/golang-stacktrace/golang-stacktrace
	获取当前执行点的栈信息
*/
//Package errors provides ability to annotate you regular Go errors with stack traces.
func getStackTrace() string {
    stackBuf := make([]uintptr, 50)
    length := runtime.Callers(3, stackBuf[:])
    stack := stackBuf[:length]

    trace := ""
    frames := runtime.CallersFrames(stack)
    for {
    	frame, more := frames.Next()
    	trace = trace + fmt.Sprintf("\n\tFile: %s, Line: %d. Function: %s",
    		frame.File, frame.Line, frame.Function)
    	if !more {
    		break
    	}
    }
    return trace
}
```

**生成错误示例：**

```go
/*
	模拟不同场景产生不同错误，使用入参instruction进行选择
 */
func CreateWithDiffError(instruction string) error {
    switch instruction {
    case "typeErr":
	//上下文添加备注
    	return &TypeError{"Lack of energy.", reflect.TypeOf(TypeError{}), getStackTrace()}
    case "sizeErr":
    	//上下文添加时间信息
    	return &SizeError{"time:" + time.UnixDate, reflect.TypeOf(SizeError{})}
    case "userErr":
    	//上下文添加用户
    	return &UserError{"UserErr with selfContext: @pixelpig.",
    		reflect.TypeOf(UserError{})}
    default:
	return errors.New("UnknownError")
    }	
}
```
----
上面提到，因为Error的接口实现是通过指针实现的，所以可以通过.(type)进行类型断言，针对不同错误类型进行处理。  

**我们来看下类型断言场景：**

```go
func ParseErr(err error) {
    if err != nil {
    	switch e := err.(type) {
    	case *TypeError:
    		log.Printf("ErrType[%v] Context[%s]\n, Trace[%s]\n", e.Type, e.context, e.trace)
    		//TODO: Handle 类型错误
    		break
    	case *UserError:
    		log.Printf("ErrType[%v] Context[%s]\n", e.Type, e.context)
    		//TODO: Handle 用户错误
    		break
    	case *SizeError:
    		log.Printf("ErrType[%v] Context[%s]\n", e.Type, e.context)
    		//TODO: Handle 容量错误
    		break
    	default:
    		log.Println(err)
    	}
    }
}
```
> err.(type)，这里的type是自定义Error结构体反射获取的类型，与生成```Error```处赋值的Type属性是两个含义。 

**主程序：**

```go
var (
    TYE = "typeErr"
    SE = "sizeErr"
    UE = "userErr"
)

func main() {
    typeErr := CreateWithDiffError(TYE)
    seErr := CreateWithDiffError(SE)
    timeErr := CreateWithDiffError(UE)
    
    ParseErr(typeErr)
    ParseErr(seErr)
    ParseErr(timeErr)
}
```
**程序输出：**
```go
2020/02/02 13:08:25 ErrType[main.TypeError] Context[Lack of energy.]
, Trace[
	File: D:/goProject/src/HelloGo/basic/ErrorHandle/ErrorDemo.go, Line: 19. Function: main.main
	File: D:/Go1.12/src/runtime/proc.go, Line: 200. Function: runtime.main
	File: D:/Go1.12/src/runtime/asm_amd64.s, Line: 1337. Function: runtime.goexit]
2020/02/02 13:08:25 ErrType[main.SizeError] Context[time:Mon Jan _2 15:04:05 MST 2006]
2020/02/02 13:08:25 ErrType[main.UserError] Context[UserErr with selfContext: @pixelpig.]
```
可以看到第一个自定义类型打印出了程序执行的栈信息。

### Wraping：error嵌套

如果不想通过**自定义error**来实现内嵌信息，在Go1.13以上版本，提供了一个新的错误包装方式，通过扩展**fmt.Errorf**函数，加一个```%w```来生成一个可以包装的错误，通过这种方式，我们可以创建一个嵌套Error。

#### 示例：

```go
/*
	包装错误
 */
func WrapErr(err error) error{
    return fmt.Errorf("Wrap with shell: %v", err)
}
```
经过```Errorf```的包装会返回一个新的```Error```，其中“Wrap with shell”是包装的内容。  
#### 输出如下：

```go
2020/02/02 17:20:38 Wrap with shell: UserError contex: UserErr with selfContext: @pixelpig.
```

如果要还原解开错误，**Go1.13** 的```errors```提供了一个**errors.Unwrap(w)** 方法，返回原始错误，即被嵌套的那个**error** ，支持多次还原，直到返回nil。



## 总结
**Go语言**不像**Java**有**Exception**异常和**JVM堆栈**环境，没有帮我们封装执行栈的数据到内置**error**中，它建议程序员在错误发生处尽早处理，不倾向于把错误往上层抛，所以如果要方便追溯错误在程序的位置，可以通过生成自定义错误，植入函数栈的位置。  

如果遵循Go的推荐实践，大部分情况希望我们在错误发生处进行handle，尽早处理，那么假如仅仅需要一些上下文信息，比如时间，用户等。这种情况可以使用**自定义error**，或者使用Go自带包装方式，存储我们额外需要的字段，如上述的**UserError**。

## 参考链接
**Custom errors in golang and pointer receivers**  
https://stackoverflow.com/questions/50333428/custom-errors-in-golang-and-pointer-receivers  

**Error Handling In Go, Part I**  
https://www.ardanlabs.com/blog/2014/10/error-handling-in-go-part-i.html  
**Error Handling In Go, Part II**  
https://www.ardanlabs.com/blog/2014/11/error-handling-in-go-part-ii.html  


//TODO：
https://dave.cheney.net/2015/01/26/errors-and-exceptions-redux