---
title: "简述 Java线程池以及使用"
date: 2019-07-15

categories:
- Java
- 并发
tags:
- 线程
showSocial: false
---
线程池可以避免频繁创建销毁线程，提高性能。常见如数据库的连接池。
<!--more-->

#### 创建线程的几种方式
1. 继承**Thread**，重写**run()**方法
2. 实现**Runnable**接口，实现**run()**方法
3. 实现**Callable**接口，实现**Call()**方法
4. 使用**线程池**，并向其提交任务**task**，其内部自动创建线程调度完成。
####上述对比：
一般来说，使用第一种和第二种比较普遍，考虑到Java不支持多继承，通常使用第二种，实现Runnable接口创建线程。
然而，假设创建线程进行一些复杂的处理，比如需要执行后做出反馈，这时候可以使用实现**Callable**方式创建线程。
About Callable的返回值，可以使用Future或者FutureTask来获取。

----
#### Future
它是一个接口，包含方法：
**boolean cancel();**
尝试取消任务，返回ture包括（任务已完成，任务成功取消）
**boolean isCancelled();**
区别于上一种方法这个更像是一个未完成状态（完成任务之前的取消），不包括已完成
**boolean isDone();**
判断任务是否完成
**get();**
获取执行结果，如果任务在执行该方法会阻塞，直到有返回值
**get(long timeout,TimeUnitunit);**
在有限时间内阻塞得到返回值，否则返回null

#### FutureTask:
底部实现**RunnableFuture**接口，而**RunnableFuture**继承了**Future**和**Runnable**
所以**FutureTask**同样具备上述5个方法。
参考部分源码：

```
/**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Callable}.
     *
     * @param  callable the callable task
     * @throws NullPointerException if the callable is null
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
可以看到**FutureTask**可以传入参数**Callable**或者**Runnable，Result**作为构造方法，区别在于：
1. 前者调用**get()**方法获取的是**Callable**的**call()**返回值
2. 后者传入参数**Runnable**和**Result**,如果执行成功，返回**Result**

### Futuretask和Callable实现创建线程的Demo

```
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class CallableThread {
	public static void main(String[] args) {
		ThreadCall tc = new ThreadCall();
		FutureTask<String> result = new FutureTask<>(tc);
		Thread t = new Thread(result);
		t.start();
		while (true) {
			if (result.isDone()) {
				try {
					System.out.println("Call执行结束:"+result.get());
					break;
				} catch (InterruptedException | ExecutionException e) {
					e.printStackTrace();
				}
			}
		}
	}
}

class ThreadCall implements Callable<String>{

	@Override
	public String call() throws Exception {
		return "为Java打Call！";
	}
	
}
```
执行结果：
Call执行结束:为Java打Call！

----
#### 线程池：
***线程池可以避免频繁创建销毁线程，提高性能。常见如数据库的连接池。***
在**JUC**（java.util.concurrent）中，Java提供了一个工具类**Executors**用于创建线程池[多种类型]，返回一个执行器，再将**Runnable**或者**Callable**提交到执行器中[详见demo]。
##### 线程池的几个类型：
 - **newCachedThreadPool(int nThreads)**
接受任务才创建线程，如果某个线程空闲超过60秒，则会被撤销。
 - **newFixedThreadPool()**
 固定线程数量的线程池，如果线程池数量超过nThreads,新任务会放在任务队列中，如果某个线程终止了，另一个线程会来取代它。
 - **newSingleThreadExecutor()**
  只有一个线程的线程池，依次执行任务 
 - **newScheduledThreadPool(int corePoolSize)**
 指定按照设置的时间周期执行任务，池中线程数量根据参数**corePoolSize**决定，即时有线程空闲状态也不会撤销。

#### Demo:

```
package com.dd.code.ThreadByCallable;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class TestThreadPool {
	
	public static void main(String[] args) {
		//创建线程池（5个线程）
		ExecutorService pool = Executors.newFixedThreadPool(5);
		ThreadPoolDemo tpd = new ThreadPoolDemo();
		
		//为线程池分配10个任务
		for (int i = 0; i < 10; i++) {
			pool.submit(tpd);
		}
		//关闭线程池(等待任务执行完成才关闭)，区别于shutdownnow立即关闭
		pool.shutdown();
	}
	
}

class ThreadPoolDemo implements Runnable{

	private int i = 0;
	
	@Override
	public void run() {
		System.out.println("当前线程号："+Thread.currentThread().getName() + "，i = " + i++);
	}
	
}

```
执行结果：

```
当前线程号：pool-1-thread-3，i = 2
当前线程号：pool-1-thread-5，i = 3
当前线程号：pool-1-thread-1，i = 0
当前线程号：pool-1-thread-2，i = 1
当前线程号：pool-1-thread-3，i = 6
当前线程号：pool-1-thread-5，i = 5
当前线程号：pool-1-thread-4，i = 4
当前线程号：pool-1-thread-3，i = 8
当前线程号：pool-1-thread-2，i = 7
当前线程号：pool-1-thread-1，i = 9

```
可以看到，尽管分配10个任务给线程池，然而我们初始化5个固定线程的线程池，所以线程池执行的所在进程号不会超过5，而且可以看出在**newFixedThreadPool**中线程的调度是随机的。

----
#### 线程池多任务计算
假如我们要使用线程池来执行相应计算，并且取得返回值要如何实现呢？
通常可以使用Future或者FutureTask以结合Callable来实现。
#### 思路：

 1. 创建所需线程池，返回执行器[**ExecutorService**]**executor**
 2. 使用**FutureTask**构造方法传入需要计算的**Callable**实例化创建**FutureTask**
 3. 调用执行器**executor**的方法**executor.execute(futuretask);**
 4. 创建**`List<FutureTask>`**将执行的**futuretask**添加到List中，方便后续取值。
####示例：
**下面模拟一个计算延迟操作，我们来看看多线程和单线程执行的效率对比**

```
package com.dd.code.ThreadByCallable;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;
import java.util.concurrent.RejectedExecutionException;

public class MutilThreadTask {
	// 线程（内部类）实现Callable接口
	static class MyCallable implements Callable<Long> {
		private long uid;

		public MyCallable(long uid) {
			super();
			this.uid = uid;
		}

		@Override
		public Long call() throws Exception {
			return Work(uid);
		}

		public long Work(long uid) {
			try {
				//模拟查询操作延迟
				Thread.sleep(100);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			return uid * 2;
		}

	}

	public static void main(String[] args) {
		// 多线程累加list
		List<Long> uidList = new ArrayList<Long>();
		for(int i = 0; i < 100; i++){
			uidList.add(10L);
		}
		
		long t1  = System.currentTimeMillis();
		//使用单线程进行计算
		System.out.println("累加结果:" + SumWithOneThread(uidList));
		System.out.println("常规计算时间:"+(System.currentTimeMillis()-t1));
		
		long t2  = System.currentTimeMillis();
		//创建10个线程的线程池参与运算
		System.out.println("累加结果:" + SumWithMutilThread(uidList,10));
		System.out.println("线程池计算时间:"+(System.currentTimeMillis()-t2));
	}
	
	static Long SumWithOneThread(List<Long> uidList){
		Long result = 0L;
		for (Long l : uidList) {			
			result += DoubleIt(l);
		}
		return result;
	}
	
	static Long DoubleIt(Long l){
		try {
			//模拟延迟计算
			Thread.sleep(100);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return l*2;
	}
	
	static Long SumWithMutilThread(List<Long> uidList,int THREAD_NUM) {
		// 创建线程池
		ExecutorService executor = Executors.newFixedThreadPool(THREAD_NUM);
		// 关于Future和Futuretask可以参考http://blog.csdn.net/zmx729618/article/details/51596414
		// 创建任务列表
		List<FutureTask<Long>> ftlist = new ArrayList<FutureTask<Long>>();
		// MutilThreadTask mt = new MutilThreadTask();
		// 逐个执行任务并且添加到任务列表
		for (Long uid : uidList) {
			/*
			 * 实例化内部类MyCallable方法
			 * 1.把内部类设为静态. 
			 * 2.通过外部类.实例化内部类，MyCallable callable = mt.new MyCallable(uid);
			 */
			MyCallable callable = new MyCallable(uid);
			FutureTask<Long> futuretask = null;
			try {
				futuretask = new FutureTask<Long>(callable);
				executor.execute(futuretask);
			} catch (RejectedExecutionException e) {
				//Re:处理进入线程池失败的情况，也就是说，不仅获取线程资源失败，并且由于等待队列已满，甚至无法进入队列直接 失败
				e.printStackTrace();
			}
			ftlist.add(futuretask);
		}
		// 遍历任务列表，把完成任务取出来获取数据
		long totalResult = 0L; // 初始化计算数据量
		for (FutureTask<Long> ft : ftlist) {
			Long result = null;
			try {
				result = ft.get();
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
			if (result != null) {
				totalResult += result;
			}
		}
		// 关闭执行器
		executor.shutdown();
		return totalResult;
	}

}

```
执行结果：

```
累加结果:2000
常规计算时间:10035
累加结果:2000
线程池计算时间:1020
```
#### 效率对比
果然，使用线程池创建多个线程计算效率快了许多，但是使用多线程就一定快吗?
我们稍作修改一下方法的参数，只给FutureTask分配5个任务[区别于之前的100个任务]，并且将模拟计算时间修改为1毫秒[区别于之前的100毫秒]，再看看执行结果。

```
累加结果:100
常规计算时间:6
累加结果:100
线程池计算时间:24
```
发现使用线程池的多线程计算反而更慢了，可以初步断定，如果任务量不是很大，计算量不是很复杂的情况下，多线程并不是首先，相反**线程池还需要耗费资源去维护池中的线程**，所以考虑多线程的使用场景，还要权衡一下任务量。