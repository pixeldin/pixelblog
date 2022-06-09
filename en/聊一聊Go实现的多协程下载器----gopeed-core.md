---
title: "聊一聊Go实现的多协程下载器: gopeed-core"
draft: true
date: 2021-06-29
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/wood-chunk.jpg
categories:
- Go
- 并发
- 网络编程
tags:
- 文件读写
- HTTP header
- errgroup

metaAlignment: center
---

之前在Github社区偶然发现一个Go实现的下载器，作为个人学习的一个参照模板，这里要感谢一下作者monkeyWie(网名)的开源项目[gopeed-core](github.com/monkeyWie/gopeed-core)。
<!--more-->

## 前言
之前在Github社区偶然发现一个Go实现的下载器，由于此前对于这方面没有研究，抱着了解其内部实现原理的好奇心，拉取了相关开源代码，期间进行调试与分析，作为个人学习的一个参照模板，这里要感谢一下作者monkeyWie(网名)的开源项目[gopeed-core](github.com/monkeyWie/gopeed-core)。

## 思路
在初步浏览梳理相关代码逻辑之后，大概总结下下载器的主要流程如下：
1. 发起请求获取下载链接的文件名、文件大小、是否支持“断点续传”等状态。
2. 支持断点续传？
   - 支持，则依据并发数量，为每个协程独立分配下载任务区间
   - 不支持，单协程同步下载
3. 启动并发下载，由HTTP请求header获取文件字节区间，分而治之。

## 相关技术点
下面先列举项目一些引用到的基础知识，熟悉用法之后再来聊聊项目中的具体应用实现。
### ErrGroup
在Go原生的```sync```包中有一个常用的结构，```sync.WaitGroup```，在并发编程中经常会使用到，其中的```wg.Wait()```用于通知调用处进行等待，等待所有期待的任务执行```wg.Done()```，方便我们实现一个阻塞等待的框架。  
在它之后衍生出一个```sync.ErrGroup```包，可以在```wait()```处返回任务执行返回的```error```
- 等待子任务完成
- catch子任务err并决定做何处理  

在将一个任务分解为多块并分治的场景十分适合这种设计模型，类似于```MapReduce```或者称为‘scatter and gather’。 比如分段下载、分页查找等。
> when error occurs, context cancelled, so use select will fine, “scatter and gather”(分治)  

**使用方法:**  
errGroup通常伴随一个errCtx上下文出现，用于通知同一个组内的协程，处理逻辑可以由代码控制。
```go
// 创建一个errGroup，以及一个errCtx上下文
eg, errCtx := errgroup.WithContext(context.Background())
```
使用```eg.Go()```启动，传入签名为```func() error```的函数作为参数：  
```go
eg.Go(func() error {
    // 任意业务错误
	return errors.New("egError")
})
```

**内部分析：**  
观察源码可以知道出错时会调用```g.cancel()```，对应着上面返回err上下文的```errCtx.Done()```
```go
// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()
        // 如果任务返回err，显式调用g绑定的cancel()函数
		if err := f(); err != nil {
			g.errOnce.Do(func() {
			    // 这里的err不会有shadowed，即只上报第一个错误
				g.err = err
				if g.cancel != nil {
				    // 触发errCtx.Done()
					g.cancel()
				}
			})
		}
	}()
}
```
**示例demo：**
```go
const (
	CON = 10
)

func main() {
    var eg *errgroup.Group
	eg, errCtx := errgroup.WithContext(context.Background())
	for i := 0; i < CON; i++ {
		i := i
		eg.Go(func() error {
			if i == 3 {
				return errors.New(fmt.Sprintf("Mock err: %d", i))
			}
			select {
			// 让Done分支判断在Mock err出现后命中, 等待1s
			case <-time.After(time.Duration(1) * time.Second):
			case <-errCtx.Done():
			    fmt.Println("meet err in job:", i)
				return errCtx.Err()
			}
			fmt.Println(i)
			return nil
		})
	}

	// wait for done or err occurs
	if err := eg.Wait(); err == nil {
		log.Print("all looks good.")
	} else {
		log.Printf("errors occurs: %v", err)
	}
	return
}
```
**输出：**  
可以看到，在组内i等于3的协程抛出错误之后，组内其他协程```errCtx.Done()```分支都命中了，所以它们直接返回errGroup捕获的**第一个错误**。
```
2021/06/24 07:24:55 meet err in job: 9
2021/06/24 07:24:55 meet err in job: 1
2021/06/24 07:24:55 meet err in job: 4
2021/06/24 07:24:55 meet err in job: 2
2021/06/24 07:24:55 meet err in job: 0
2021/06/24 07:24:55 meet err in job: 7
2021/06/24 07:24:55 meet err in job: 8
2021/06/24 07:24:55 meet err in job: 5
2021/06/24 07:24:55 meet err in job: 6
2021/06/24 07:24:55 errors occurs: Mock err: 3
```

### HTTP Header参数
#### HTTP/1.1
区别于HTTP1.0，HTTP/1.1除了提供了长连接的功能之外，还有一个升级点就是在HTTP1.1之后开启了断点续传功能，即请求端可以通过Range标签只请求资源的某个部分。
#### Header参数
|名词解释|含义|
|---|---|
|Range|设置请求文件字节区间，例：Range：bytes=0-1023，表示前1024个字节|
|Content-Range|Content-Range：bytes 0-1023/2047，服务器返回Header，表示此次请求区间，以及文件总大小。|
|状态码：206|使用Range请求文件区间之后，服务端一般会返回206表示支持断点续传|
|Content-Disposition|正文描述，本例用于获取文件名(也可从URL连接进行解析)|

**示例demo：**  
- 启动本地文件服务器
```go
// 启动本地文件服务器，打开127.0.0.1:8899可以看到共享文件列表
http.Handle("/", http.FileServer(http.Dir("D:\\测试下载\\serverFile")))
if err := http.ListenAndServe(":8899", nil); err != nil {
	fmt.Println("err:", err)
}
```
- 相关常量
```go
// basic/downloader/mdl/constansts.go
const (
	HttpCodeOK             = 200
	HttpCodePartialContent = 206

	HttpHeaderRange              = "Range"
	HttpHeaderContentLength      = "Content-Length"
	HttpHeaderContentRange       = "Content-Range"
	HttpHeaderContentDisposition = "Content-Disposition"

	HttpHeaderRangeFormat = "bytes=%d-%d"
)
```
- 解析目标文件信息(获取文件大小/是否支持断点续传)
```go
const (
	DOWNLOAD_URL = "http://127.0.0.1:8899/demo.zip"
	SAVE_PATH    = "D:\\测试下载\\download"
	CON          = 8
	RETRY_COUNT  = 5
)

// ...

// 构造请求并设置header
req, err := buildReq(ctx, reqURL)
if err != nil {
	return nil, err
}

// 只访问一个字节，测试资源是否支持Range请求
req.Header.Set(mdl.HttpHeaderRange, fmt.Sprintf(mdl.HttpHeaderRangeFormat, 0, 0))
resp, err := http.DefaultClient.Do(req)

// 解析是否支持断点续传/文件大小
if resp.StatusCode == mdl.HttpCodePartialContent {
	// 支持断点下载
	res.Range = true
	// 从Content-Range:bytes 0-0/137783533 获取大小
	totalSize := path.Base(resp.Header.Get(mdl.HttpHeaderContentRange))
}
```

### 文件读写
这部分是下载过程，主要是文件的操作
- 文件创建  
    在上一个初始化下载链接之后拿到文件大小了，所以可以预创建文件并分配总占用字节数。
    ```go
    func touch(fileName string, size int64) (file *os.File, err error) {
        // 创建文件
    	file, err = os.Create(fileName)
    	if size > 0 {
    	    // 分配大小
    		err := os.Truncate(fileName, size)
    		if err != nil {
    			return nil, err
    		}
    	}
    	return
    }
    ```
    
- 文件写入  
    在并发数大于0以及文件协议支持断点续传的前提下，开启多协程并发下载，这里引出了一个困惑。  
    **核心问题: 多协程并发写同一个文件，如何避免加锁提高效率？**  

    笔者在刚开始拉取项目代码的时候，作者已经优化过并提供了最终方案了，我当时“先入为主”地忽视了这个问题，之后在看**commit**记录**和issue列表**的时候才觉察最初这部分作者是做了互斥锁的。  

    **部分代码如下：**
    ![互斥锁代码](https://user-images.githubusercontent.com/15009201/58263942-94c1fa00-7daf-11e9-8aaa-d232a88fe147.png)

    之后回想其实并发操作文件并无影响，因为我们在刚开始就创建文件并分配总大小，后续子协程只需要根据自己负责的文件字节下标区间去写入就好了，这里与写顺序无关，因此其实不需要加锁。（笔者后来也找作者再次确认这个猜想）  
    
    最终作者实现的代码如下，  
    **定义每个文件切分的块，下标用于向服务器申请文件区间：**
    ```go
    type Chunk struct {
    	Status Status       // 文件块下载状态
    	Begin  int64        // 开始下标
    	End    int64        // 结束下标
    	Downloaded int64    // 当前文件块已下载字节, 用于文件追加或者出错断点续传
    }
    ```
    **定义文件块数组并分配区间：**
    ```go
    var (
		// 切分文件块
		chunks []*mdl.Chunk
		// 切分块数
		ckTol int
	)
	// 支持切分
	if res.Range {
	    // 并发8个协程下载
		ckTol = 8
		chunks = make([]*mdl.Chunk, ckTol)
		partSize := res.TotalSize / int64(ckTol)
		for i := 0; i < ckTol; i++ {
			var (
				begin = partSize * int64(i)
				end   int64
			)
			if i == (ckTol - 1) {
				end = res.TotalSize - 1
			} else {
				end = begin + partSize - 1
			}
			ck := mdl.NewChunk(begin, end)
			chunks[i] = ck
		}
	} else {
		ckTol = 1
		// 单连接下载
		chunks = make([]*mdl.Chunk, ckTol)
		chunks[0] = mdl.NewChunk(0, 0)
	}
    ```
    **下载细节：**
    ```go
    func fetch(ctx context.Context, res *mdl.Resource,
	    file *os.File, chks []*mdl.Chunk, c int, doneCh chan error) {
	    // 使用errgroup捕获异常并通知外部doneCh通道
    	eg, _ := errgroup.WithContext(ctx)
    	for i := 0; i < c; i++ {
    	    // 注意闭包写法
    		i := i
    		eg.Go(func() error {
    		    // 并发下载
    			return fetchChunk(ctx, res, file, i, chks)
    		})
    	}
    
    	go func() {
    		// error from errgroup
    		err := eg.Wait()
    		// 关闭文件
    		file.Close()
    		// 接收fetchChunk()内部状态
    		doneCh <- err
    	}()
    	return
    }
    
    func fetchChunk(ctx context.Context, res *mdl.Resource, file *os.File, index int, chks []*mdl.Chunk) (err error) {
    	ck := chks[index]
    	req, err := buildReq(ctx, DOWNLOAD_URL)
    	if err != nil {
    		return err
    	}
    	var (
    		client = http.DefaultClient
    		buf    = make([]byte, 8192)
    	)
    	/**************重试区间开始**************/
    	// 根据是否分块下载设置header
    	//	- 根据当前chunk设置文件区间到header
    	//	- 判断请求返回status
    	//		- 失败: 少于5次则做重试
    	//		- 成功:
    	//				根据offset, 把buf写入文件
    	//				- 成功: return, 通知外部
    	//				- 失败: 重试少于5次, 返回重试
    	for i := 0; i < RETRY_COUNT; i++ {
    		var resp *http.Response
    		if res.Range {
    			req.Header.Set(mdl.HttpHeaderRange,
    				fmt.Sprintf(mdl.HttpHeaderRangeFormat, chks[index].Begin+ck.Downloaded, chks[index].End))
    		} else {
    			// 单连接重试没有断点续传
    			ck.Downloaded = 0
    		}
    
    		// 获取字节区间
    		if err := func() error {
    			resp, err = client.Do(req)
    			if err != nil {
    				return err
    			}
    			if resp.StatusCode != mdl.HttpCodeOK && resp.StatusCode != mdl.HttpCodePartialContent {
    				return errors.New(fmt.Sprintf("%d,%s", resp.StatusCode, resp.Status))
    			}
    			return nil
    		}(); err != nil {
    			continue
    		}
    
    		// 从body提取buf写入文件, 重置重试标识
    		i = 0
    		retry := false
    		retry, err = func() (bool, error) {
    			defer resp.Body.Close()
    			for {
    				n, err := resp.Body.Read(buf)
    				if n > 0 {
    				    // 每次写入从已下载位置开始写，保证字节顺序
    					_, err := file.WriteAt(buf[:n], ck.Begin+ck.Downloaded)
    					if err != nil {
    						// 文件出错不重试
    						return false, err
    					}
    					// 记录已下载, 用于断点续传
    					ck.Downloaded += int64(n)
    				}
    				if err != nil {
    					// err from read
    					if err != io.EOF {
    						return true, err
    					}
    					break
    				}
    			}
    			// success exit
    			return false, nil
    		}()
    		if !retry {
    			break
    		}
    	}
    	/**************重试区间结束**************/
    
    	// 通知外部
    	return
    }
    
    ```
    其实代码逻辑相对简单，只是加多了错误重试所以读起来有点绕，关键点在于```ck.Downloaded```字段的维护更新，无论是正常下载还是失败重试，都需要严格保证```ck.Downloaded```当前值的准确性。
   
## 代码分析
笔者对原版代码进行调整和部分核心功能的抽取，以求快速理解其下载思路，作者使用了较常用的工厂和适配器模式，后续如果有机会可以展开分析，再次感谢作者monkeyWie提供的轮子。

## 参考地址
**Coordinating goroutines — errGroup**  
https://levelup.gitconnected.com/coordinating-goroutines-errgroup-c78bb5d80232  
**monkeyWie/gopeed-core**  
https://github.com/monkeyWie/gopeed-core