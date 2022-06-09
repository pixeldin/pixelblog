---
title: "聊一聊Go编写的命令行工具类"
date: 2020-02-20
draft: true

categories:
- Go
- 命令行编程
- 后端
tags:
- flag
- cmd
showSocial: false
---

Go原生在flag包提供了一个命令行工具类，它可以让我们执行类似命令行的赋参操作，经常被运用于工具类，特别是数据处理过程，可以方便我们进行参数可视化注解。
<!--more-->

## 命令行工具

### 原生flag包
Go原生在flag包提供了一个命令行工具类，它可以让我们执行类似命令行的赋参操作，经常被运用于工具类，特别是数据处理过程，可以方便我们进行参数可视化注解。

flag包提供了多个常用类型的赋值方法，如```String, Int, Bool, Float64, Duration```等。
- 通过flag.XXXType()函数可以对参数名称，默认值，描述进行定义
- ```flag.parse()```用于指定程序开始解析传入的命令行变量
- 参数-h可以调用flag内置的```usage()```方法， 把参数描述打印出来  

下面列举一个简单的例子。

#### 用法示例：

```go
func main() {

	//声明接收变量, 注意返回的是个指针类型
	wordParam := flag.String("word", "defaultValue", "Desc param 1 for something.")
	numParam := flag.Int("number", 0, "Desc param 2 for integer number.")

	//an existing var declared elsewhere in the program
	ok := AllRight()

	flag.BoolVar(&ok, "ok", false, "Desc value for boolean.")

	flag.Parse()

	log.Printf("Flag for wordParam: %s", *wordParam)
	log.Printf("Flag for wordParam: %d", *numParam)
	log.Printf("Flag for inner bool value : %v", ok)
}

func AllRight() bool {
	return true
}
```
我们通过command line 传参给可执行程序，**输出：**

```shell
$ ./tool.exe -word=hello -number=6
2020/03/18 10:59:49 Flag for wordParam: hello
2020/03/18 10:59:49 Flag for wordParam: 6
2020/03/18 10:59:49 Flag for inner bool value : false

```

如果使用```-h/--help```命令可以调出参数提示，**输出：**

```shell
$ ./tool.exe -h
Usage of E:\Code\GoWork\src\HelloGo\basic\flag\tool.exe:
  -number int
        Desc param 2 for integer number.
  -ok
        Desc value for boolean.
  -word string
        Desc param 1 for something. (default "defaultValue")
```

----
### flag分级用法
在实际应用环境，调用目标可能有多个，有时我们需要多个命令，多个参数联合起来，用于调用不同的方法，类似于参数调用子命令的参数，  
如：```./cmd foo -a="a" -b=1```或者```./cmd bar -c="c" -d=2```
#### Subcommands
原生flag包提供了一个子命令构造方式，```NewFlagSet``` 用于返回子命令的flag，**示例如下：**

我们定义一个解析子命令方法：

```go
func SubCommands() {
	//param1: 命令参数, param2: 错误退出码
	fooCmd := flag.NewFlagSet("foo", flag.ExitOnError)
	//声明子命令fc, fc2
	fc := fooCmd.String("fc", "default of fc", "foo sub value1")
	fc2 := fooCmd.String("fc2", "default of fc2", "foo sub value2")

	barCmd := flag.NewFlagSet("bar", flag.ExitOnError)
	//声明子命令bc
	bc := barCmd.String("bc", "default of bc", "bar sub b")

	if len(os.Args) < 2 {
		fmt.Println("expected 'foo' or 'bar' subcommands")
		fooCmd.Usage()
		barCmd.Usage()
		os.Exit(1)
	}

	//参数逻辑调用, 使用分支语句
	switch os.Args[1] {
	case "foo":
		fooCmd.Parse(os.Args[2:])
		log.Printf("fc of foo: %s", *fc)
		log.Printf("fc2 of foo: %s", *fc2)
	case "bar":
		barCmd.Parse(os.Args[2:])
		log.Printf("bc of bar: %s", *bc)
	default:
		fmt.Println("expected 'foo' or 'bar' subcommands")
		os.Exit(1)
	}

}
```
执行命令行提示：

```shell
$ ./tool.exe
expected 'foo' or 'bar' subcommands
Usage of foo:
  -fc string
        foo sub value1 (default "default of fc")
  -fc2 string
        foo sub value2 (default "default of fc2")
Usage of bar:
  -bc string
        bar sub b (default "default of bc")
```
命令调用示例：

```shell
$ ./tool.exe foo -fc="fc value" -fc2="fc22222"
2020/03/18 12:54:16 fc of foo: fc value
2020/03/18 12:54:16 fc2 of foo: fc22222
```
### 问题延申：
上述方式借助Go的内置flag实现了命令传参，但是有个问题，如上面提到，“**在实际应用环境，调用目标可能有多个，有时我们需要多个命令**”，对每个调用链都起一个switch分支去写可能十分吃力，我们自己能不能对调用链执行封装，尝试实现类似于flag的功能呢？

----
### 自定义分支调用
最近在预览前辈的代码块中，发现一个巧妙的实现方式，在Go中，函数是可以作为参数传入的，我们可以利用这一特性建立一个方法链数组，利用全局函数变量根据入参进行调用，**类似于面向对象语言里面的多态，本质上是相同类型的不同表现。**

#### 程序实例：
- 先定义一个cmd结构体，封装了指令/执行方法/提示
```go
/*
	自定义方法调用
 */
//封装命令结构, 指令/执行方法/提示
type cmd struct {
	Name    string
	Process func(args ...string)
	Usage   func() string
}

//全局指令数组
var all []*cmd
```
- 根据cmd的规范建立多个实现类，如下面的例子，有wind， sun两个实现方式  

**wind实现：**
```go
//可变参数作为可选入参
func Process(args ...string)  {
	if len(args) < 2 {
		logrus.Error(Usage())
		return
	}

	//Do something
	logrus.Infof("Wind fly from %s, level %s.", args[0], args[1])
}

//参数提示
func Usage() string {
	return "Hint: wind <N/S/W/E> <level>"
}
```

**sun实现：**
```go
func Process(args ...string)  {
	if len(args) < 2 {
		logrus.Error(Usage())
		return
	}

	//Do something
	logrus.Infof("Sun %s about %s miles.", args[0], args[1])

}

//参数提示
func Usage() string {
	return "Hint: sun <rise/down> <range>"
}
```

至此， 不同指令的表现就实现完了，**有点类似于面向对象语言里面的多态，本质上是相同类型的不同表现。**  
后面我们来看下在**main**方法怎么调用
```go
func init()  {
	all = append(all, &cmd{"sun", sun.Process, sun.Usage})
	all = append(all, &cmd{"wind", wind.Process, wind.Usage})
}

//全局用法
func usage() string {
	sb := new(strings.Builder)
	sb.WriteString(fmt.Sprintf("Usage: %s <command> [args...]\n", os.Args[0]))
	for _, c := range all {
		sb.WriteString("\n命令: ")
		sb.WriteString(c.Name)
		sb.WriteString(", 用法: ")
		sb.WriteString(c.Usage())
		sb.WriteString("\n")
	}
	return sb.String()
}

func main()  {
	if len(os.Args) > 1 && os.Args[1] != "--help" && os.Args[1] != "-h"{
		for _, c := range all {
			//匹配全局命令
			if c.Name == os.Args[1] {
				c.Process(os.Args[2:]...)
				return
			}
		}
		fmt.Fprintf(os.Stderr, "No match func for %s.\n", os.Args[1])
	}
	fmt.Fprintln(os.Stderr, usage())
}
```
我们利用入参的长度以及内置命令区分调用链，对符合格式的参数与全局指令切片进行匹配。  
下面是 **输出示例：**

```shell
$ ./tool.exe -h
Usage: E:\Code\GoWork\src\HelloGo\basic\flag\tool.exe <command> [args...]

命令: sun, 用法: Hint: sun <rise/down> <range>

命令: wind, 用法: Hint: wind <N/S/W/E> <level>

$ ./tool.exe a b
No match func for a.
Usage: E:\Code\GoWork\src\HelloGo\basic\flag\tool.exe <command> [args...]

命令: sun, 用法: Hint: sun <rise/down> <range>

命令: wind, 用法: Hint: wind <N/S/W/E> <level>
```
可以看到当参数不合法时会遍历打印各个子命令的用法给予提示。  
```shell
$ ./tool.exe sun
time="2020-03-18T15:23:23+08:00" level=error msg="Hint: sun <rise/down> <range>"
```

**常规调用：**
```shell
$ ./tool.exe wind N 7
time="2020-03-18T15:22:40+08:00" level=info msg="Wind fly from N, level 7."
$ ./tool.exe sun rise 30
time="2020-03-18T15:24:28+08:00" level=info msg="Sun rise about 30 miles."
```

### 小结
以上是go命令行工具的简单用法，内置的flag帮我们封装好了一下基础函数，我们也可以利用字符串处理自定义实现工具类，兼容多种场景，简单的逻辑判断也能达到flag的那种用法。

### 参考链接：
**Git project**：  
https://github.com/pixeldin/HelloGo/tree/master/basic/flag  
**Go by Example: Command-Line Subcommands**  
https://gobyexample.com/command-line-subcommands  
**Command line flag syntax**  
https://golang.org/pkg/flag/#hdr-Command_line_flag_syntax