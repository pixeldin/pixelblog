---
title: "OpenResty Notes"
date: 2022-11-13
thumbnailImagePosition: right
categories:
- Lua
- gateway
tags:
- NGINX
- OpenResty
- LuaJIT

metaAlignment: center
---

从NGINX出发，再到插件定制化，作为一个国产API网关的沉淀，OpenResty开始逐渐摆脱NGINX的影子。
<!--more-->
![openresty-home](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/bird-home.png)

# ❓特性是什么
💡 NGINX网关+插件化管理
- 特性一: 同步非阻塞
- 特性二: 动态热更新
- 特性三: 测试用例驱动完善

## 前言
从NGINX出发，再到插件定制化，作为一个国产API网关的沉淀，OpenResty开始逐渐摆脱NGINX的影子，形成自己的生态体系，在API网关、软WAF等领域被广泛使用，不再局限于NGINX最初的功能。

## 🔨如何解决

### 项目模块

#### 子项目:
- perl编写的文档
- resty目录: LuaJIT
- 测试框架**test-nginx**
- 调试工具: 火焰图\代码格式化

#### 包管理工具:

- OPM
- LuaRocks

#### 执行阶段:
![OpenResty执行阶段](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/openresty_process.png)
- `set_by_lua：设置变量；`
- `rewrite_by_lua：转发、重定向等；`
- `access_by_lua：准入、权限等；`
- `content_by_lua：生成返回内容；`
- `header_filter_by_lua：应答头过滤处理；`
- `body_filter_by_lua：应答体过滤处理；`
- `log_by_lua：日志记录。`

### Lua 对比 LuaJIT
#### Lua

- 借助虚拟机，编译字节码→机器码
- **使用Lua C API调用C函数**

#### LuaJIT
![OpenResty Lua嵌入](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/openresty-architecture.png)

- 在上面基础上增加热代码检测, 对于频繁调用代码开启缓冲, 自己转成机器码
    
    > **所谓 LuaJIT 的性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式退到 Lua 解释器的解释执行模式**
    > 
- **使用FFI调用C函数**

### 什么是FFI
Foreign Function Interface 外部函数接口，**直接在Lua代码调用外部C函数。**

### 与C模块合作
- lua-nginx-module  
    集中了C函数API，使用Lua C API实现
- lua-resty-core  
    LuaJIT通过FFI方式实现C函数API，在**非解释模式**下执行更加高性能。

### 避免使用NYI(Not Yet Implemented)
> 💡 当 JIT 编译器在当前代码路径上遇到它不支持的操作时，便会退回到解释器模式

- 避免`string.dump`
- 使用`ngx.re.find()`替换`string.find(模式匹配)`
- 避免使用`unpack()`使用下标访问代替

### 使用`resty -j v -e` 执行并且检测是否支持JIT

```lua
root@pixelpig:/usr/local/openresty# resty -j v -e 'for i=1, 1000 do
> local newstr, n, err = ngx.re.gsub("hello, world", "([a-z])[a-z]+", "[$0,$1]", "i")
> end'
[TRACE   1 regex.lua:1081 loop]
[TRACE   2 (1/10) regex.lua:1116 -> 1]
[TRACE   3 (1/21) regex.lua:1084 -> 1]
```

### 事件模型
**cosocket**: coroutine + socket
![event model](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/Lua%20event%20mechanism.png)

### 数据共享

- 共享变量：`$val` 仅支持字符串
- ngx.ctx: 在同一个请求共享，**随请求结束销毁，如果发生重定向，则会丢失**，可以借鉴开源库[lua-resty-ctxdump](https://github.com/tokers/lua-resty-ctxdump/blob/master/README.md)
- 使用模块变量，在相同的woker内部共享，但是**要避免写，防止竞争问题**
- 共享字典`shared dict` 可以在多个woker共享，性能高，只支持字符串，需要预配置
    
    ```lua
    lua_shared_dict share_dict_xxx 10m;
    ```

### 真值与空

Lua：**除了`nil`和`false` 其他都是真值**。

OpenResty衍生的非空：`ngx.null`、`cdata:NULL`、`cjson.null` 

---

## 测试框架test::Nginx
配合测试用例以及DSL规范熟悉上手，关键字`---ONLY` \ `---LAST` \ `---SIIP` 等。

## 压测配置

`fs.file-max:` 全局最大文件打开数量

`ulimit -n:` 一个进程可以打开的文件数量

## 性能概要

### 资源分析
- top
- pidstat
- vmstat
- iostat
- sar  
> USE(utilization\saturation\errors)

### 负载分析
- perf
- systemtap
- bcc/ebpf
- 火焰图


### 避免阻塞操作

- 执行外部命令`os.execute()` 使用外部`ffi`或者`ngx.pipe`替换
- 替换磁盘i\o为异步网络，可以用`cosocket` 异步实现上报远程
- 只在init阶段执行阻塞操作，如`luasocket`

### 字符串操作

- 避免频繁创建字符串，加重GC代价，使用`Lua table`保存字符串
- 避免产生临时字符串，可使用`string.byte` → `string.char`进行转换

### table事项

- 预分配table大小，避免插入元素再自增
- table插入使用下标，尽量避免#获取table长度`t[#t + 1]`
- 使用`table.clear`清空复用`table`,可以配合`tablepool`使用

## 性能监控
- 断点、日志
- 动态追踪、调试
- Systemtap
- 火焰图
  - 横坐标：CPU分配时间占比
  - 纵坐标：函数栈深度

## 动态

### 动态调试
基于动态语言的`hook`模块进行debug

### 动态加载
> 动态，指的是程序可以在运行时、在不重新加载的情况下，去修改参数、配置，乃至修改自身的代码。OpenResty是通过脚本语言Lua来完成的⸺脚本语言的一大优势，便是运行时可以去做动态地改变。
> 

### 动态上游(`ngx.balancer`)
被动+主动健康监测

## 🔗参考资料

[简书：LuaJit](https://www.jianshu.com/p/0f968605d36d)  

[Apisix动态调试](https://apisix.apache.org/zh/blog/2022/08/19/apache-apisix-runtime-dynamic-debugging/)  

[The price of speed: Lua or LuaJIT? Etiene Dalcol - London Lua August 2017](https://www.youtube.com/watch?v=p4AzAaJ8Ick)

[OpenResty最佳实践](https://github.com/moonbingbing/openresty-best-practices)