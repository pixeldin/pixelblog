---
title: "Go协程间通信之生产者-消费者模型"
date: 2021-01-18T03:40:05+08:00
categories:
- Go
- 并发
- 计算机
tags:
- 生产者消费者
- 管道通信
- Concurrent


<!-- coverImage: /img/city.jpg -->
metaAlignment: center
---


不要通过共享内存来通信(互斥锁同步)，而要用通信来共享内存。
<!--more-->

### 通信原则  
Go开发有一个经常提及的原则：
> 不要通过共享内存来通信(互斥锁同步)，而要用通信来共享内存。  
### 前言
在其他模式的开发语言中，比如Java有个常见的生产者-消费者模式，通过多个线程池与多个```BlockingQueue```进行交互，如```LinkBlockedQueue```， ```ArrayBlockedQueue```等 ，由于队列内部通过锁机制帮我们集成了同步的功能，程序业务层不需要关心多线程对队列的竞争，所以可以放心的使用。  
而来到Go这边，由于```channel```天生具备同步的特性，结合上面提到的通信原则，也可以较简单的运用于**生产者-消费模型**。

### 示例
1. 创建一个共享队列作为生产者消费者连接的管道

```go
var (
    //通信管道大小
    QUEUE_SIZE            = conf.OptiInt("ext.queueSize", 5)
    //生产者并发度
	PRODUCE_SIZE            = conf.OptiInt("ext.prodSize", 3)
	//消费者并发度
	CONSUME_SIZE            = conf.OptiInt("ext.consSize", 3)
)

//共享队列
msgQueue := make(chan []byte, QUEUE_SIZE)
```

2. 消费端业务

```go
/*
*   CNum标识不同消费者的序号，由外部传入
*/
func Consume(CNum int, msg chan []byte)  {
	for value := range msg {
		logrus.Infof("# Consumer CNum.%d, take cake with value: %s.", CNum, string(value))
		//Add time costing
		time.Sleep(200)
	}
}
```

3. 生产端业务  
我们使用Go的```time```包集成的Tick方法，用来简单示例一个周期调度器，每个生产者每1秒触发一生产数据的操作。

```go
/*
*   msg为共享队列，signal是终止生产信号，
*   在Go中函数可以作为参数传递，因此把
*   job作为自定义具体业务操作
*/
func PeriodJob(msg chan []byte, signal <-chan int, job func()) bool {
	for {
		tk := time.Tick(1 * time.Second)
		select {
		case <-signal:
			Info("Receive Produce stop signal.")
			return true
		case <-tk:
		    //定期调度
			job()
		}
	}
}

//参数列表： 生产者序号/数据管道/信号管道/匿名函数作为job
func Produce(PNum int, msg chan []byte, signal <-chan int)  {
	PeriodJob(msg, signal, func() {
		cake := []byte(fmt.Sprintf("cake, @tag: PNum.%d", PNum))
		Infof("## Producer push a %s", cake)
		msg <- cake
	})
}
```

4. 调度main方法

```go
func main() {
    Info("Start main..., time: %v", time.Now())
    defer Info("Main job done!")
    
    //生产者,消费者并发度
    var costSize, prodSize = model.CONSUME_SIZE, model.PRODUCE_SIZE
    //终止生产信号
    ring := make(chan int)
    //Consumer job
    for i := 0; i < costSize; i++ {
    	go worker.Consume(i, msgQueue)
    }
    
    //Producer job
    for i := 0; i < prodSize; i++ {
    	go worker.Produce(i, msgQueue, ring)
    }
    //给生产者时间输出数据
    time.Sleep(3 * time.Second)
    //终止生产
    close(ring)
    
    /*
    	Try to idle
    	注意: 如果协程处理时间大于主协程, 提前退出有可能任务处理中断，
    	为了程序简单化，此处不做异步通知，给予消费者协程时间，
    	保证消费结束再退出。
     */
    time.Sleep(5 * time.Second)
}
```

5. 输出
```go
time="2020-01-09 11:51:09" level=info msg="Start demo..., time: 2020-01-09 11:51:09.0422629 +0800 CST m=+0.002928201"
time="2020-01-09 11:51:10" level=info msg="## Producer push a cake, @tag: PNum.1"
time="2020-01-09 11:51:10" level=info msg="# Consumer CNum.1, take cake with value: cake, @tag: PNum.1."
time="2020-01-09 11:51:10" level=info msg="## Producer push a cake, @tag: PNum.0"
time="2020-01-09 11:51:10" level=info msg="# Consumer CNum.0, take cake with value: cake, @tag: PNum.0."
time="2020-01-09 11:51:10" level=info msg="## Producer push a cake, @tag: PNum.2"
time="2020-01-09 11:51:10" level=info msg="# Consumer CNum.2, take cake with value: cake, @tag: PNum.2."
time="2020-01-09 11:51:11" level=info msg="## Producer push a cake, @tag: PNum.0"
time="2020-01-09 11:51:11" level=info msg="# Consumer CNum.3, take cake with value: cake, @tag: PNum.0."
time="2020-01-09 11:51:11" level=info msg="## Producer push a cake, @tag: PNum.1"
time="2020-01-09 11:51:11" level=info msg="# Consumer CNum.4, take cake with value: cake, @tag: PNum.1."
time="2020-01-09 11:51:11" level=info msg="## Producer push a cake, @tag: PNum.2"
time="2020-01-09 11:51:11" level=info msg="# Consumer CNum.1, take cake with value: cake, @tag: PNum.2."
time="2020-01-09 11:51:12" level=info msg="Receive Produce stop signal."
time="2020-01-09 11:51:12" level=info msg="Receive Produce stop signal."
time="2020-01-09 11:51:12" level=info msg="Receive Produce stop signal."
time="2020-01-09 11:51:17" level=info msg="Main job done!"
```



