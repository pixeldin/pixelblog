---
title: "Lua Basic Tip"
date: 2022-11-02
thumbnailImagePosition: right
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/glue.png
categories:
- Lua
- gateway
tags:
- NGINX
- OpenResty
- LuaJIT

metaAlignment: center
---

Lua作为一种胶水语言，具备动态类型，通过整合已有的高级组件构建新的应用。
<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/Lua-glue.png)

### 前言
Lua作为一种胶水语言，具备动态类型，即运行时检测，只有值才有类型。
适用场景：
> Lua语言支持组件化的软件开发方式，通过整合已有的高级组件构建新的应用。这些组件通常是通过C/C++等编译型强类型语言编写的，Lua语言充当了整合和连接这些组件的角色。

## 基础概念
### 字符串:

内化：两个完全一样的 Lua 字符串在 Lua 虚拟机中只会存储一份。每一个 Lua 字符串在创建时都会插入到 Lua 虚拟机内部的一个全局的哈希表中。 这意味着

1. 创建相同的 Lua 字符串并不会引入新的动态内存分配操作，所以相对便宜（但仍有全局哈希表查询的开销），
2. 内容相同的 Lua 字符串不会占用多份存储空间，
3. 已经创建好的 Lua 字符串之间进行相等性比较时是 `O(1)` 时间度的开销，而不是通常见到的 `O(n)`.

> 相同内容的字符串只会被保存一份，因此 Lua 字符串之间的相等性比较可以简化为其内部存储地址的比较。这意味着 **Lua 字符串的相等性比较总是为 O(1). 而在其他编程语言中，字符串的相等性比较则通常为 O(n)，即需要逐个字节（或按若干个连续字节）进行比较**
> 

### table表：
**通常实现为一个哈希表、一个数组、或者两者的混合。具体的实现为何种形式，动态依赖于具体的 table 的键分布特点。**

```lua
local corp = {
    web = "www.google.com",   --索引为字符串，key = "web",
                              --            value = "www.google.com"
    telephone = "12345678",   --索引为字符串
    staff = {"Jack", "Scott", "Gary"}, --索引为字符串，值也是一个表
    100876,              --相当于 [1] = 100876，此时索引为数字
                         --      key = 1, value = 100876
    100191,              --相当于 [2] = 100191，此时索引为数字
    [10] = 360,          --直接把数字索引给出
    ["city"] = "Beijing" --索引为字符串
}

print(corp.web)               -->output:www.google.com
print(corp["telephone"])      -->output:12345678
print(corp[2])                -->output:100191
print(corp["city"])           -->output:"Beijing"
print(corp.staff[1])          -->output:Jack
print(corp[10])               -->output:360
```

#### 导出table结构
通常可以通过如下方式打印table结构，即dump table
```lua
-- debug dump for Lua table
local function dump(o)
    if type(o) == 'table' then
       local s = '{ '
       for k,v in pairs(o) do
          if type(k) ~= 'number' then k = '"'..k..'"' end
          s = s .. '['..k..'] = ' .. dump(v) .. ','
       end
       return s .. '} '
    else
       return tostring(o)
    end
 end
```

```lua
-- dump table with inspect 
local inspect = require('inspect')
core.log.info("# Pixel-debug table content: ".. inspect(table))
```

> 不要在 Lua 的 table 中使用 nil 值，**如果一个元素要删除，直接 remove，不要用 nil 去代替**
。
> 

## 元表

元方法的重载`__tostring`\`__index`\`__call`

```lua
local version = {
   major = 1,
	minor = 1,
	patch = 1
}

version = setmetatable(version, {
	__tostring = function(t)
		return string.format("%d.%d.%d", t.major, t.minor, t.patch)
	end
})
print(tostring(version))
```

## 面向对象
```lua
-- Meta class
Shape = {area = 0}
-- 基础类方法 new
function Shape:new (o,side)
   o = o or {}
   setmetatable(o, self)
   self.__index = self
   side = side or 0
   self.area = side*side;
   return o
end
-- 基础类方法 printArea
function Shape:printArea ()
   print("面积为 ",self.area)
end

-- 创建对象
myshape = Shape:new(nil,10)
myshape:printArea()

Square = Shape:new()
-- 派生类方法 new
function Square:new (o,side)
   o = o or Shape:new(o,side)
   setmetatable(o, self)
   self.__index = self
   return o
end

-- 派生类方法 printArea
function Square:printArea ()
   print("正方形面积为 ",self.area)
end

-- 创建对象
mysquare = Square:new(nil,10)
mysquare:printArea()

Rectangle = Shape:new()
-- 派生类方法 new
function Rectangle:new (o,length,breadth)
   o = o or Shape:new(o)
   setmetatable(o, self)
   self.__index = self
   self.area = length * breadth
   return o
end

-- 派生类方法 printArea
function Rectangle:printArea ()
   print("矩形面积为 ",self.area)
end

-- 创建对象
myrectangle = Rectangle:new(nil,10,20)
myrectangle:printArea()
```

## 函数

- 参数传递：默认传值, `table`类型：**传引用**
- `…`表示变长
    
    > **LuaJIT 2 尚不能 JIT 编译这种变长参数的用法，只能解释执行。所以对性能敏感的代码，应当避免使用此种形式。**
    > 
- 返回值，函数有多个返回值可以**用括号来表示只取第一个值，但是只能解释执行。**

## 弱表

```lua
-- __mode的值是 k\v，那就意味着这个table的键\值为弱引用
setmetatable(tb, {__mode = "v})
```

通过弱表声明可以避免垃圾回收泄漏未引用对象。

## 模块

### [String](https://moonbingbing.gitbooks.io/openresty-best-practices/content/lua/string_library.html)

> 由于 `string.byte` 只返回整数，而并不像 `string.sub` 等函数那样（尝试）创建新的 Lua 字符串， 因此使用 `string.byte` 来进行字符串相关的扫描和分析是最为高效的，尤其是在被 LuaJIT 2 所 JIT 编译之后。
> 

## LuaRocks
包管理工具

### String

> 由于 `string.byte` 只返回整数，而并不像 `string.sub` 等函数那样（尝试）创建新的 Lua 字符串， 因此使用 `string.byte` 来进行字符串相关的扫描和分析是最为高效的，尤其是在被 LuaJIT 2 所 JIT 编译之后。
> 


## LuaJIT
`just-in-time`，Lua语言的即时编译器。
> 即时编译器会将频繁执行的代码编译成机器码缓存起来，下次调用时将直接执行机器码。相比原生逐条执行虚拟机指令效率更高。而对于那些只执行一次的代码仍然逐条执行。

## 参考链接
- **Openresty最佳实践**  
https://github.com/moonbingbing/openresty-best-practices/  
- **《Lua程序设计》**  
- **Lua教程**  
https://www.runoob.com/lua/lua-object-oriented.html  
- **LuaJIT Introduce**  
  - https://www.jianshu.com/p/0f968605d36d  
  - https://www.youtube.com/watch?v=p4AzAaJ8Ick  
- **Lua编程技巧**  
https://blog.codingnow.com/cloud/Luatips  