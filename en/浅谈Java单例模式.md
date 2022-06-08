---
title: "浅谈 Java单例模式"
date: 2019-08-11

categories:
- Java
- 设计模式
tags:
- 后端
showSocial: false
---

相信在设计模式中有一个经常提到的概念：**单例模式**，为什么它经常出现在面试话题中，因为它的应用场景十分广泛。
<!--more-->

#### 使用场景：
比如

 - 数据库连接池，作为数据库的缓存，避免频繁连接关闭数据库，
 - Java线程池，控制管理线程。
 - log4j日志记录，由始至终记录着运行日志。


#### 定义：
> 保证系统中一个类只有**一个实例**，而且必须**自己创建自己**的唯一实例，该实例易于外界访问，从而方便对实例个数的控制并节约系统资源。

----
### 创建单例模式的几种方式以及比较

####  1. 饿汉模式

```
/*
 1. 饿汉模式：
 2. 优点：线程安全
 3. 缺陷：性能低/加载类就初始化单例/不适合需要外部传入参数配置的单例模式
 */
public class SingletonHungry {
	
	private static final SingletonHungry instance = new SingletonHungry();
	
	public static SingletonHungry getInstance() {
		return instance;
	}
		
	public static void main(String[] args) {		
		
		SingletonHungry s1 = SingletonHungry.getInstance();
		SingletonHungry s2 = SingletonHungry.getInstance();
		System.out.print("饿汉模式实例对比:");
		//true
		System.out.println(s1.getInstance()==s2.getInstance());
	}	
	
}
```
由于饿汉模式在类内部创建实例，所以它是线程安全，正式它在类内部就静态加载，所以它不能从外部传入参数配置。
具体来看看懒汉模式.
#### 2.  懒汉模式

```
package com.dd.code.singleton;

/*
 * 懒汉模式
 * 优点：简单/对比饿汉，加载此单例可以外部传入配置
 * 缺陷：线程不安全
 */
public class SingletonLazy {

	private static SingletonLazy instance;
	
	/*	
	 * 配置成员conf（假设必须传入conf该单例才可以加载）
	 * 	这不能在类中优先初始化 private static final SingletonHungry instance = new SingletonHungry();
	 */
	private static String conf;
	
	
	//外部传入属性配置
	public static void setConf(String conf) {
		SingletonLazy.conf = conf;
	}
	
	public static SingletonLazy getInstance() {
		if (instance == null) {
			instance = new SingletonLazy();
		}
		return instance;
	}

	
	public static void main(String[] args) {
		SingletonLazy.setConf("配置文件优先");
		SingletonLazy s1 = SingletonLazy.getInstance();
		SingletonLazy s2 = SingletonLazy.getInstance();
		System.out.print("懒汉模式实例对比:");
		//true
		System.out.println(s1.getInstance()==s2.getInstance());
	}
	
}

```
对比饿汉模式，懒汉模式可以传入必要配置再手动实例化，但是由于手动实例化，则需要考虑线程安全问题。

####  3. 线程安全懒汉模式

```
/*
 *  线程安全懒汉加载
 *  优点：线程安全/可以外部传配置
 *  缺陷：代价较高，创建单例只需要第一次保证线程安全就好，不需要每次都同步
 *  优化解决：SingletonDoubleCheck
 */
public class SingletonThreadSafe {
	private static SingletonThreadSafe instance;
	
	//加了同步关键字synchronized
	private synchronized SingletonThreadSafe getInstance() {
		if (instance == null) {
			instance = new SingletonThreadSafe();
		}
		return instance;
	}
	
	public static void main(String[] args) {
		SingletonThreadSafe s1 = new SingletonThreadSafe();
		SingletonThreadSafe s2 = new SingletonThreadSafe();
		System.out.print("线程安全懒汉加载实例对比:");
		//true
		System.out.println(s1.getInstance()==s2.getInstance());
	}
}

```
加了同步关键字synchronized保证了线程安全，但是它的性能就降低了，而且其实**创建单例只需要第一次保证线程安全就好，不需要每次都同步。**

所以引入了新的优化，好像很厉害的<u>双重锁检测模式</u>
####  4. 双重锁检测单例

```

/*
 * 双重锁单例模式（较复杂）
 * 优点：可传入配置/对比SingletonThreadSafe性能优化，只有在实例为空（第一次实现同步）
 * 为什么要第二次判断instance是否为空，因为把synchronized放里层的话，
 * 有可能有多个线程进入了*临界区*，synchronized只能保证临界区每次由一个线程执行而已，
 * 二次检测可以让其他线程下次不初始化，防止冗余情况
 */
public class SingletonDoubleCheck {

	/*
	 * volatile关键字：
	 * 要知道，instance = new SingletonDoubleCheck();不是原子性操作，
	 * 虽然volatile关键字不能保证原子性，但是可以禁止指令重排
	 * 保证了instance = new SingletonDoubleCheck() 这一行的有效执行顺序 
	 */
	private volatile static SingletonDoubleCheck instance;
	
	
	private static SingletonDoubleCheck getInstance() {
		if (instance == null) {
			//非临界区
			synchronized (SingletonDoubleCheck.class) {
				//*临界区*
				if (instance == null) {
					instance = new SingletonDoubleCheck();
				}
			}
		}
		return instance;
	}
	
	public static void main(String[] args) {
		SingletonDoubleCheck s1,s2;
		s1 = SingletonDoubleCheck.getInstance();
		s2 = SingletonDoubleCheck.getInstance();
		System.out.print("双重锁单例加载实例对比:");
		//true
		System.out.println(s1==s2);
	}
	
}
```
双重锁模式可以说是性能和线程安全的折中，它保证线程安全又保证不需要每次都控制同步，第一个判断if (instance == null)用来拦截已经创建的线程。
主要复杂的是从**非临界区到临界区**的情况，即当未创建实例的时候：要知道**synchronized**关键字其实只能保证每次一个线程执行**修饰代码块**，并不能保证只有一个线程，假设有超过一个线程进入临界区，此时如果线程一执行instance = new SingletonDoubleCheck()，则线程2下一次会根据第二个if (instance == null)判断是否再次创建实例，所以第二个if其实相当于一个flag标记，它**巧妙的避免了多个线程创建多个实例**。

----
这种方式有点烧脑，官方推荐还有另一种创建方式，静态内部类模式，**比较推荐使用的一种**。
####  5. 静态内部类

```
/*
 * 静态内部类 
 */
public class SingletonNested {

	private static class SingletonHolder{
		public static final SingletonNested HOLDER_INSTANCE = new SingletonNested();
	}

	public static SingletonNested getInstance() {
		return SingletonHolder.HOLDER_INSTANCE;
	}
	
	public static void main(String[] args) {
		SingletonNested s1, s2;
		s1 = SingletonNested.getInstance();
		s2 = SingletonNested.getInstance();
		System.out.print("静态内部类实例对比:");
		//true
		System.out.println(s1==s2);
	}
}
```
静态内部类创建单例的优点：
> 1. 由jvm保证线程安全，不需要用到同步控制，性能较高;
> 2. 因为区别于懒加载把instance作为静态内部成员，所以类加载时不会实例化instance 只有getInstance调用才会初始化。


参考链接：

 - http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/

 - http://blog.csdn.net/lovelion/article/details/7420888