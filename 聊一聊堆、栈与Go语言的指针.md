---
title: "聊一聊堆、栈与Go语言的指针"
date: 2020-04-16
<!-- thumbnailImagePosition: left
thumbnailImage: /img/go-context.jpg -->
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-cmd.png -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/go-cmd.png
categories:
- Go
- 操作系统
tags:
- 指针
- 堆栈
showSocial: false
---
堆、栈在计算机领域是亘古不变的热门话题，归根结底它们都是操作系统层面的内存划分。
<!--more-->

### 前言
堆、栈在计算机领域是亘古不变的热门话题，归根结底它们和编程语言无关，都是操作系统层面的内存划分，后面尝试简单地拆开这几个概念，谈谈我对它们的理解。

### 栈
每个函数中每个值在栈中都是独占的，不能在其他栈中被访问。每个方法片(function frame)都有一个自己的独享栈，这个栈的生命周期随着方法开始结束诞生与消逝，在方法结束时候会被释放掉，较之于堆，栈的优势是比较轻量级，随用随弃，存活期跟随着函数。

### 堆
通俗的讲，假如说栈是各个函数的一栋私人住宅，堆就是一个大型的人民广场，它可以被共享。堆作为一个全局访问块，它的空间由GC(拆迁大队)管理作。
> The heap is not self cleaning like stacks, so there is a bigger cost to using this memory. Primarily, the costs are associated with the garbage collector (GC), which must get involved to keep this area clean.

翻译过来，区别于栈在函数调用结束时候就释放掉，堆不会自动释放，堆空间的释放主要来自于垃圾回收操作。  
### GC（Garbage collection）
垃圾回收, 垃圾回收具有多种策略，一般来说，每一个存在于堆中，但不再被指针所引用的变量，都会被回收掉。“These allocations put pressure on the GC because every value on the heap that is no longer referenced by a pointer, needs to be removed.”由于垃圾回收涉及内存操作，往往需要考虑许多因素，所幸经过漫长演变，前人种树，有些编程语言垃圾回收策略已经足够强大（Java，Go），我们大部分时候不需要去干涉内存清理，把这部分工作交给底层调度器。  

分配到堆内存有个弊端是它会为下一次GC增加压力，好处是可以被其他栈所共享，下列情景编译器会倾向于将它放在堆中存储：
- 尝试申请一个较大的结构体/数组
- 变量在一定的时间内还会被使用
- 编译期间不能确认大小的变量申请

### 指针
指针，本质上和其他类型一样，只不过它的值是内存地址(引用)，我的理解是内存块的门牌号。有句经常被提及的话：
> 什么时候该使用指针取决于什么时候要分享它。  
Pointers serve one purpose, to share a value with a function so the function can read and write to that value even though the value does not exist directly inside its own frame.

指针是为了让变量在不同函数方法块（栈区间）之间分享，并且提供变量读写操作。


----
结合上述的理解，指针指的是内存地址，堆是共享模块，指针是为了共享同一块内存片。在Go语言中，所有传参都是值传递，指针也是通过传递指针的值。

### 程序实例 指针共享
#### 主函数

```go
func main() {
	var v int = 1
	fmt.Printf("# Main frame: Value of v:\t\t %v, address: %p\n", v, &v)
	PassValue(v, &v)
	fmt.Printf("# Main frame: Value of v:\t\t %v, address: %p\n", v, &v)
}
```
#### 子函数

```go
func PassValue(fv int, addV *int) {
	// fv 的地址只属于该函数, 由该函数栈分配
	fmt.Printf("# Func frame: Value of fv:\t\t %v, address: %p\n", fv, &fv)
	//本次修改只在该函数生效
	fv = 0
	fmt.Printf("# Func frame: Value of fv:\t\t %v, address: %p\n", fv, &fv)

	/*
	 *	根据main函数传入的全局地址, 对指针执行操作外部是可见的,
	 *  因为改指针操作的都是同一个内存块的内容
	 */
	*addV++
	fmt.Printf("# Func frame: Value of addV:\t %v, address: %p\n", *addV, addV)
}
```

#### 输出:

```go
# Main frame: Value of v:		 1, address: 0xc000054080
# Func frame: Value of fv:		 1, address: 0xc0000540a0
# Func frame: Value of fv:		 0, address: 0xc0000540a0
# Func frame: Value of addV:	 2, address: 0xc000054080
# Main frame: Value of v:		 2, address: 0xc000054080
```
可以看到传递指针在子函数里面操作的都是**同一个地址（0xc000054080）**,所以在子函数退出时候对v的改变在主函数是可见的。  
而传进子函数的fv所在地址已经处于子函数的管辖栈，随着函数结束，该栈会被释放。

---------

### 栈逃逸 
1. **访问外部栈**  
指的是变量在函数执行结束时候，没有随着函数栈结束生命，值超出函数（栈）的调用周期，逃到堆去了（通常是一个全局指针），能被外部所共享，可以通过go自带的工具来分析。下面是栈逃逸的一个栗子。

    ```go
    //go:noinline
    func CreatePointer() *int  {
    	return new(int)
    }
    ```
    **分析**
    ```go
    $ go build -gcflags "-m -m -l" escape.go
    # command-line-arguments
    .\escape.go:9:12: new(int) escapes to heap
    .\escape.go:9:12:       from ~r0 (return) at .\escape.go:9:2
    ```
    可以看到提示, ```return new(int)```这个语句把new指针返回给调用方，这个随着CreatePointer函数结束的时候仍然有外部引用到这个指针，已经超出了该函数栈的范围，所以编译器提示它将分配到堆中去。

2. **编译未确定**  
关于栈逃逸，还有另外一种情景会发生，再看一个栗子：

    ```go
    func SpecifySizeAllocate()  {
    	buf := make([]byte, 5)
    	println(buf)
    }
    
    func UnSpecifySizeAllocate(size int)  {
    	buf := make([]byte, size)
    	println(buf)
    }
    ```
    **分析**
    
    ```go
    $ go build -gcflags "-m -m" escape.go
    # command-line-arguments
    .\escape.go:5:6: can inline SpecifySizeAllocate as: func() { buf := make([]byte, 5); println(buf) }
    .\escape.go:10:6: can inline UnSpecifySizeAllocate as: func(int) { buf := make([]byte, size); println(buf) }
    .\escape.go:6:13: SpecifySizeAllocate make([]byte, 5) does not escape
    .\escape.go:11:13: make([]byte, size) escapes to heap
    .\escape.go:11:13:      from make([]byte, size) (non-constant size) at .\escape.go:11:13
    
    ```
    观察这两个函数，两个buf的生存期都只在函数里面，好像都不会逃逸，然而根据分析结果，```UnSpecifySizeAllocate()```这个函数却产生了栈逃逸，这是为什么呢？  
    可以看到，分析提示“non-constant size”/“没有具体大小”，这是因为，**编译器并不能在编译阶段知道size的值,所以没法在函数栈里面分配确切大小的区间给buf，如果编译器遇到这种未确定的大小分配，会把他分配到堆中去。** 这也解释了为什么```SpecifySizeAllocate()```函数没有产生逃逸。

----
### 后记：
经过时间的演变编译器已经足够智能，堆栈的申请分配可以放心交给它们去做，大部分业务代码并不需要过度考虑变量的分配，这里仅仅是尝试刨析程序变量在内存中的划分，理解一些概念，要知道堆栈分析只是性能调优其中的一种方式。  

当然，这里不仅仅是性能问题，在参数传递中，使用值拷贝还是使用指针，最好要结合这个变量未来的作用域而决定。

**Go**自带一些工具方便我们分析底层的实现，在遵循前人的建议下，堆栈的分析可能更适合处于业务代码完成之后的优化阶段中，前期为了保证代码的功能和可读性，程序猿应该首选专注于实现，当后面遇到性能瓶颈了，尝试从堆栈分配处优化可能才是要考虑的，毕竟有个原则叫做不要过早优化。


### 参考链接:
**官档：How do I know whether a variable is allocated on the heap or the stack**  
https://golang.org/doc/faq#stack_or_heap  
**Ardan labs 四连干货(推荐):**   
https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-escape-analysis.html  
**Go: Should I Use a Pointer instead of a Copy of my Struct?**  
https://medium.com/a-journey-with-go/go-should-i-use-a-pointer-instead-of-a-copy-of-my-struct-44b43b104963  
**Memory : Stack vs Heap**  
https://www.gribblelab.org/CBootCamp/7_Memory_Stack_vs_Heap.html