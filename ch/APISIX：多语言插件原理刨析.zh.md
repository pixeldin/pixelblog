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

## 前言

## 原理刨析
### 进程间通信(IPC)
Socket，作为内核中的一个套接字结构体，可用于进程间通信。
### Plugin Runner
- 使用`APISIX_LISTEN_ADDRESS`指定生成`socket`接口的目录，提供用于插件进行外部通信，如不指定，在`UnitTest`模式下会在默认目录生成：
  ```
  root@pixelpig:~/work/apisix# ll t/servroot/html/
  total 0
  drwxr-xr-x 1 root root 512 Jan  3 18:05 ./
  drwxr-xr-x 1 root root 512 Jan  3 18:05 ../
  -rw-r--r-- 1 root root  72 Jan  3 18:05 index.html
  srw-rw-rw- 1 root root   0 Jan  3 18:05 nginx.sock=
  ```



## 语言对比


## 参考链接
- **Openresty最佳实践**  
https://github.com/moonbingbing/openresty-best-practices/  
- **Lua编程技巧**  
https://blog.codingnow.com/cloud/Luatips  