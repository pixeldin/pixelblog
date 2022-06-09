---
title: "Java反射机制和代理模式的应用"
date: 2019-03-15
draft: true

categories:
- Java
- 反射
- 代理模式
tags:
- 后端
showSocial: false
---

RPC：远程过程调用
<!--more-->

## Java反射机制
引入反射机制，先从.class对象说起，先看一个简单的实例化操作

```
	@Test
	public void test3() {
		Student s = new Student();
		Class<Student> clazz = (Class<Student>) s.getClass();
		System.out.println(clazz);
	}
```

其中 s是运行时类对象，clazz是运行时类
### 运行时类
> 当我们编译一个Java源程序是，创建一个类，通过编译（javac），生成.class文件，之后使用java.exe给每个类加载.class文件，当new多个实例（运行时类对象）时候，都指向缓存区的.class(每一个运行时类只加载一次)把.class文件加载到内存，就是一个运行时类，它在缓存区只存一份。

在Object类中具有一个public final Class getClass()方法，该方法为所有类所拥有。凭借它可以获得一个类的**.class**对象。


### 作用
> Java反射机制作为动态语言的关键，反射机制通过**.class**文件，借助于Reflection API取得任何类的内部信息，通过它可以直接操作任意对象的内部属性及方法。

1. 传统创建对象的方式是，**Student s = new Student();**
而通过class实例，也可以创建运行时实例的对象。
比如：

```
Class<Student> clazz = Student.class;

// 创建Clazz对应的运行时类Student对象(注意student要有权限足够的无参构造器，否则要明确调用对应有参构造方法并且传递参数)
Student s = clazz.newInstance();
```

2.  通过class实例，获取完整类结构、属性、方法、构造器、包、父类、接口、内部类等一系列成员。
 常用方法：
  -  **clazz.getMethods();**返回类中权限为public方法

 -  **clazz.getDeclaredMethods();**返回类中所有方法
 

3. 通过1创建 实例之后，调用相应方法。

#### 示例代码：
##### 新建Human类

```
package com.dd.code.reflect;

public class Human<T> {
	public double height;
	private String DNA = "defalut";
	public void addHeight() {
		System.out.println("长高了");
	}
}

```

##### Student类继承自Human

```
package com.dd.code.reflect;

public class Student extends Human<String>{
	private int age;
	public String name;
	public int getAge() {
		return age;
	}
	public void setAge(int age) {
		this.age = age;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	
	
	
	public Student() {
		super();
	}
	public Student(int age, String name) {
		super();
		this.age = age;
		this.name = name;
	}
	
	public void show() {
		System.out.println("我是个学生");		
	}
	
	public void display(String language) {
		System.out.println("我讲"+language);
	}
	
	@Override
	public String toString() {
		return "Student [age=" + age + ", name=" + name + "]";
	}
	
	
	
}

```

##### 创建TestReflection.java,测试常用方法

```
	@Test
	public void test3() {
		Student s = new Student();
		// s是运行时类对象，clazz是运行时类
		Class<Student> clazz = (Class<Student>) s.getClass();
		System.out.println(clazz);//class com.dd.code.reflect.Student

	}
```
##### 通过反射获取class实例的4种方法：

```
	@Test
	public void test4() throws ClassNotFoundException {
		//1. 调用运行时类本身的.class属性
		Class clazz1 = Student.class;
		System.out.println("通过运行时类获取class实例" + clazz1.getName());

		//2. 运行时对象获取
		Student s = new Student();
		Class<Student> clazz2 = (Class<Student>) s.getClass();
		System.out.println("通过运行时对象获取class实例" + clazz2.getName());

		//3. Class的静态方法获取(源码实际调用4.的类加载器加载)	
		String className = "com.dd.code.reflect.Student";
		Class class4 = Class.forName(className);
		//造对象 class4.newInstance()		
		System.out.println("通过Class静态方法forName获取class实例：" + class4.getName());
		
		//4.通过类加载器
		ClassLoader classLoader = this.getClass().getClassLoader();
		Class clazz5 = classLoader.loadClass(className);
		
		//验证只存在一个class实例
		System.out.println(clazz1==class4);//true
		System.out.println(clazz1==clazz5);//true
		
	}
```

其中第3种其实是调用了第4种，如果知道类的全类名，通过它创建**class**实例，则要通过**Class**类的静态方法forName()获取，并且抛出**ClassNotFoundException**

----
获得class对象之后，接下来看看如何通过它创建类的实例：
### class对象创建类的实例

```
Class<Student> clazz = Student.class;

// 创建Clazz对应的运行时类Student对象(注意student要有权限足够的无参构造器，否则要明确调用对应有参构造方法并且传递参数)
Student s = clazz.newInstance();
System.out.println(s);//Student [age=0, name=null]
```
通过调用Class对象的newInstance()方法要注意：

 1. Student类必须有一个无参数的构造器
 2. 类的构造器需要足够的访问权限
查看**java.lang.Class.newInstance()**源码

```
@CallerSensitive
    public T newInstance()
        throws InstantiationException, IllegalAccessException
    {
        if (System.getSecurityManager() != null) {
            checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
        }

        // NOTE: the following code may not be strictly correct under
        // the current Java memory model.

        // Constructor lookup
        if (cachedConstructor == null) {
            if (this == Class.class) {
                throw new IllegalAccessException(
                    "Can not call newInstance() on the Class for java.lang.Class"
                );
            }
            try {
                Class<?>[] empty = {};
                final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
                // Disable accessibility checks on the constructor
                // since we have to do the security check here anyway
                // (the stack depth is wrong for the Constructor's
                // security check to work)
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedAction<Void>() {
                        public Void run() {
                                c.setAccessible(true);
                                return null;
                            }
                        });
                cachedConstructor = c;
            } catch (NoSuchMethodException e) {
                throw (InstantiationException)
                    new InstantiationException(getName()).initCause(e);
            }
        }
        ...
    }
```

其中的**newInstance()**方法对调用类的构造方法进行检测，如果不存在无参构造器，则会抛出**Can not call newInstance() on the Class for java.lang.Class**异常。

##### getDeclaredConstructor(Class … parameterTypes)构造获取实例化对象
那如果Student不存在无参构造器，那么可以用public Constructor<T>[] getConstructors()返回有参构造方法

```
//调用有参构造方法
Constructor<Student> con = clazz.getConstructor(int.class,String.class);
Student s2 = (Student) con.newInstance(24,"pixelpig");
System.out.println(s2);//Student [age=24, name=pixelpig]
```

#### 调用类内部成员方法：
**Object invoke(Object obj, Object …  args)**
相关：
1. Object 对应原方法的返回值，若原方法无返回值，此时返回null
2. 若原方法若为静态方法，此时形参Object obj可为null
3. 若原方法形参列表为空，则Object[] args为null
4. 若原方法声明为private,则需要在调用此invoke()方法前，显式调用方法对象的setAccessible(true)方法，将可访问private的方法。

示例：

```
// 反射调用方法
Student s = clazz.newInstance();
Method m1 = clazz.getMethod("show");
// 调用无参数方法
m1.invoke(s);
//调用toString
Method ts = clazz.getMethod("toString");
Object returnVal = ts.invoke(s);
System.out.println(returnVal);
```

#### 获取内部成员

```
@Test
public void getFromClazz(){
	Class clazz = Student.class;
	//getFields只能获取到运行时类中声明和父类为public的属性
	System.out.println("获取类内属性,包括继承父类的public成员==================");
	Field[] fields = clazz.getFields();
	for (int i = 0; i < fields.length; i++) {
		System.out.println(fields[i]);
		//public java.lang.String com.dd.code.reflect.Student.name
		//public double com.dd.code.reflect.Human.height
	}
	//获取本身已经声明的属性，包括私有（如果要操作，要设置setAccessible为true）
	System.out.println("获取已声明的属性==================");
	Field[] fields2 = clazz.getDeclaredFields();
	for (int i = 0; i < fields2.length; i++) {
		//权限名
		System.out.print(Modifier.toString(fields2[i].getModifiers())+":");
		//属性名
		System.out.println(fields2[i].getName());
		//private:age
		//public:name
	}
}
```
以上是关于反射的几个常用方法，接下来引入**动态代理**

----
## 动态代理
首先引入代理模式：
> In proxy pattern, a class represents functionality of another class,proxy is another Structural design pattern which works ‘on behalf of’ or ‘in place of’ another object in order to access the later.
> 简单理解为：在代理模式中，此类代表其他功能类的，借助调用者实现被代理对象（实际执行的实现类）的操作。

***Talk is cheap,show the code.***
### 静态代理的Demo
先看一个静态代理的例子：

```
package com.dd.code.Proxy;

/*
 * 静态代理模式
 */

//接口
interface CellPhoneFactory {
	void productPhone();
}

interface FoodFactory {
	void productFood();
}

// 被代理类，实际操作类
class GoogleFactory implements CellPhoneFactory {

	@Override
	public void productPhone() {
		System.out.println("Google工厂生产pixel");
	}

}

class KFCFactory implements FoodFactory {

	@Override
	public void productFood() {
		System.out.println("肯德基炸鸡翅。。。");
	}

}

// 代理类
class ProxyFactory implements CellPhoneFactory, FoodFactory {
	CellPhoneFactory cf;
	FoodFactory ff;

	// 传入实际代理对象
	public ProxyFactory(CellPhoneFactory cf) {
		this.cf = cf;
	}

	public ProxyFactory(FoodFactory ff) {
		this.ff = ff;
	}

	@Override
	public void productPhone() {
		System.out.println("代理类收到请求，内部开始生产手机");
		// 代理对象调用的是传入被代理对象的方法
		cf.productPhone();
	}

	@Override
	public void productFood() {
		System.out.println("代理类收到请求，内部开始生产食品");
		ff.productFood();
	}

}

public class StaticProxy {
	public static void main(String[] args) {
		GoogleFactory googleFactory = new GoogleFactory();
		ProxyFactory phoneproxy = new ProxyFactory(googleFactory);
		phoneproxy.productPhone();

		KFCFactory kfcFactory = new KFCFactory();
		ProxyFactory foodproxy = new ProxyFactory(kfcFactory);
		foodproxy.productFood();
	}
	

}

```
在上例中，

 1. 代理类**PeoxyFactory**都实现了手机工厂和食品工厂
 2. 代理类的构造方法分别方法传入了**工厂类型**。
 3. 代理类所代理的操作生产手机，生成食品，其实内部都间接调用了传入的**工厂类型**的成员方法
 4. 创建代理对象实例的时候，我们传入的不是**手机工厂和食品工厂**，而是手机工厂和食品工厂的**实现类**----谷歌生产家，和肯德基生产家
 5. 相当于调用代理对象的生产操作(**Action**），其实调用的是传入接口实现类的**Action**
所以输出结果如下：

```
代理类收到请求，内部开始生产手机
Google工厂生产pixel
代理类收到请求，内部开始生产食品
肯德基炸鸡翅。。。
```
以上是静态代理的简单示例，但是它的代码相对冗余，想象一下如果生产多种类型的产品，就要让代理类实现多种不同工厂的接口，能不能**通过反射，只传入被代理对象给代理对象，直接返回一个工厂接口**，并通过该工厂接口调用**Action**，以上说法有点抽象，

### 动态代理：
其实就是说，能否让
```
class ProxyFactory implements CellPhoneFactory, FoodFactory
```
的**implements CellPhoneFactory, FoodFactory**去掉，
相当于**不在编译的时候确定代理类，而在真正运行加载的时候，根据被代理类及其实现的接口，动态创建代理类。**
代理的特征：通过传入实现类给代理类，然后直接调用代理类的抽象方法，执行被代理类的实际方法（Action）
#### 实现动态代理：
1. 创建接口和实现类，实现类作为被代理类 
	
	```
	interface Subject{
		void action();
	}
	
	//被代理类
	class RealSubject implements Subject{
		public void action() {
			System.out.println("被代理类执行");
		}
	}
	```
 
2. 新建代理类MyInvocationHandler实现**InvocationHandler**接口
3. 创建部成员**Obj obj**作为传入被代理对象的声明 
4.  新增绑定方法**blind(Object obj)**	

	```
	Object obj;//实现接口的被代理类对象声明,相当于被代理类
	//给被代理（实际操作）对象实例化/返回代理类对象
	public Object blind(Object obj) {
		this.obj = obj;
		//根据被代理对象，制造并且返回代理类对象
		return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
				.getClass().getInterfaces(), this);
	}
	```
 
	作用：传入被代理对象**obj**（kfc工厂、Google工厂）,调用**Proxy**传入obj(kfc工厂)
4. 重写**invoke()**方法
```
//每当调用invok，实际是调用（被重写的那个）action
public Object invoke(Object proxy, Method method, Object[] args)
		throws Throwable {
	System.out.println("MyInvocationHandler/invoke");
	//method返回值
	Object returnVal = method.invoke(obj, args);
	return returnVal;
}
```

 **invoke**的作用相当于前面<u>静态代理</u>的**Action**(**ProductFood**、**ProductPhone**)。让代理对象调用的invoke其实是调用传入对象（kfc工厂、Google工厂）的内部**Action**。

#### 调用代码：
1. 实例化被代理对象
RealSubject real = new RealSubject();
2. 创建代理对象
MyInvocationHandler handler = new MyInvocationHandler();
3. 绑定被代理对象，返回实现real所在类的接口代理类对象
**real可以说是obj的实现类，即obj是Subject**，返回obj
Object obj = handler.blind(real);
4. 将obj转化为代理类对象
Subject sub = (Subject)obj;
5. 代理执行
sub.action();
###完整示例：

```
/*
 * 动态代理
 */

interface Subject{
	void action();
}

//被代理类
class RealSubject implements Subject{
	public void action() {
		System.out.println("被代理类执行");
	}
}

class MyInvocationHandler implements InvocationHandler{

	Object obj;//实现接口的被代理类对象声明,相当于被代理类
	//给被代理（实际操作）对象实例化/返回代理类对象
	public Object blind(Object obj) {
		this.obj = obj;
		//根据被代理对象，制造并且返回代理类对象
		return Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
				.getClass().getInterfaces(), this);
	}
	
	//每当调用invok，实际是调用（被重写的那个）action
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		System.out.println("MyInvocationHandler/invoke");
		//method返回值
		Object returnVal = method.invoke(obj, args);
		return returnVal;
	}
	
}

public class DynProxy {
	public static void main(String[] args) {
		//被代理对象
		RealSubject realSubject = new RealSubject();
		//创建实现InvocationHandler接口的对象
		MyInvocationHandler handler = new MyInvocationHandler();
		//调用blind,动态返回实现real所在类被实现的接口Subject
		Object obj = handler.blind(realSubject);
		Subject sub = (Subject) obj;
		//用接口对象去调，实际调用实现类(RealSubject)的action
		sub.action();		
		
	}
	
	

}
```

如果切换到静态代理的生产手机例子：
则改为

```
@Test
public void DynProxy() {
	// 创建实现InvocationHandler接口的对象
	MyInvocationHandler handler = new MyInvocationHandler();
	// 被代理对象
	GoogleFactory gf = new GoogleFactory();
	CellPhoneFactory cellPhoneFactory = (CellPhoneFactory) handler.blind(gf);
	cellPhoneFactory.productPhone();	

}
```

### 采用代理模式的优势：
> 代理对象可以在客户端和目标对象之间起到中介的作用，这样起到了中介的作用和保护了目标对象的作用。
> 其次，动态代理让Proxy类的代码量被固定下来，不会因为业务的逐渐庞大而庞大。

----
### 动态代理与AOP
> AOP面向切面编程，指的是将那些影响了多个类并且与具体业务无关的公共行为 封装成一个独立的模块（称
为切面），降低代码的耦合度。

我们就上面的例子再让动态代理与AOP进行演示：
1. 新增通用类

```
class PhoneCommon {
	public void method1() {
		System.out.println("method1：申请资金=====");
	}
	
	public void method2() {
		System.out.println("method1：样品测试=====");
	}
}
```
2. 在**MyInvocationHandler的invoke**新增切片（PhoneCommon的method1、method2）
	

	```
	//每当调用invok，实际是调用（被重写的那个）action
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {		
		PhoneCommon commonUtil = new PhoneCommon();
		//切入method1
		commonUtil.method1();
		//method返回值
		Object returnVal = method.invoke(obj, args);
		//切入method2
		commonUtil.method2();
		return returnVal;
	}
	```
这样当代理类执行被代理类的**action**,会在之前和之后执行通用类**method1()**、**method()2**

执行代码:

```
// 动态代理
@Test
public void DynProxy() {
	// 创建实现InvocationHandler接口的对象
	MyInvocationHandler handler = new MyInvocationHandler();
	System.out.println("手机生产：");
	// 被代理对象
	GoogleFactory gf = new GoogleFactory();
	CellPhoneFactory cellPhoneFactory = (CellPhoneFactory) handler.blind(gf);
	cellPhoneFactory.productPhone();

	System.out.println("\n炸鸡生产：");
	KFCFactory kf = new KFCFactory();
	FoodFactory foodFactory = (FoodFactory) handler.blind(kf);
	foodFactory.productFood();	
	
}
```

运行结果如下：

```
手机生产：
method1：申请资金=====
Google工厂生产pixel
method2：样品测试=====

炸鸡生产：
method1：申请资金=====
肯德基炸鸡翅。。。
method2：样品测试=====
```
AOP作为动态代理的运用比较广泛，常见还有日志功能、Spring的事务（Before、After、Throw ）等。