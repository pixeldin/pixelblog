---
title: "APISIX：多语言插件原理刨析"
date: 2022-11-22
categories:
- Lua
- gateway
tags:
- NGINX
- OpenResty
- LuaJIT

metaAlignment: center
---

APISIX在以Lua语言为背景开发出插件库，除此之外还支持其他语言的插件，根据官方文档，其原理是类似`side-car`的设计。
<!--more-->
<!-- ![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/Lua-glue.png) -->

# 前言

# 原理刨析
> Apache APISIX 会把 plugin runner 作为自己的一个子进程
## 进程间通信(IPC)
创建独立进程用于请求通信，要求其他语言通过socket和APISIX进行通信。（次数性能损耗比数据大小更加大，所以尽量减少通信次数。）Socket，作为内核中的一个套接字结构体，可用于进程间通信。
### 进程通信方式：
- 基于内核SYSV消息队列（开发量太大）
- 共享内存（开发量太大）
- Unix socket
- pipe IO (性能较高但是在多进程架构实现难度大)

### 序列化方式：
二进制：
- msgpack（没有主流IDL实现）
- protobuf（decode时候需要decode整个文件，无法按需decode来减少和APISIX交互次数）
- flatbuffer（向官方提供LuaJit实现）
    [Add LuaJit support](https://github.com/google/flatbuffers/pull/6584/files)
    
### APISIX Plugin Runner
- 使用`APISIX_LISTEN_ADDRESS`指定生成`socket`接口的目录，提供用于插件进行外部通信，如不指定，在`UnitTest`模式下会在默认目录生成：
  ```
  root@pixelpig:~/work/apisix# ll t/servroot/html/
  total 0
  drwxr-xr-x 1 root root 512 Jan  3 18:05 ./
  drwxr-xr-x 1 root root 512 Jan  3 18:05 ../
  -rw-r--r-- 1 root root  72 Jan  3 18:05 index.html
  srw-rw-rw- 1 root root   0 Jan  3 18:05 nginx.sock=
  ```


## WASM
对`C++\Rust`支持友好，但是对`Go`协程调度受限，WASM对`cooroutine`原生支持还在purposal阶段。


## 共享库
即将语言编译成C共享库，输出为共享对象二进制文件`(.so)` ，如`Kong`框架支持`Go`插件：


# 语言对比


# 参考链接
- **Openresty最佳实践**  
https://github.com/moonbingbing/openresty-best-practices/  
- **Lua编程技巧**  
https://blog.codingnow.com/cloud/Luatips  