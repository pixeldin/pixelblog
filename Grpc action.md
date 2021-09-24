---
title: "Grpc action"
date: 2020-03-15

categories:
- Go
- 并发
tags:
- RPC
showSocial: false
---

RPC：远程过程调用简写
<!--more-->

### HTTP发展
- HTTP1.0
- HTTP1.1 - 至今
- HTTP2.0的优势  
  (https://www.yt.com/watch?v=hNFM2pDGwKI&list=WL&index=2&t=1271s)  
    - 二进制传输
    - 缓存相同的Header，只传输改变的参数(不止重复请求被缓存起来，而且每次请求都被压缩，所以性能能显著提升。)
    - 基于TCP流式请求, no blocked

### 传统API交互：
基于RESTful的HTTP + JSON传统交互方式
弊端：
- 每个客户端，即HTTP发起端都需要人工编写
- 不支持流式处理，请求是无状态的
- 不支持全双工通信


### Protocol buffer 
- 本质上是一个协议，基于二进制传输
- **.proto** 规范文件
- 自动生成  
传输实体，客户端，服务端所有你能想到的要素，都集中定义在一个叫*.pb.go的文件里，它由.proto文件生成。  
  1. 在环境变量PATH添加两个可执行文件: ```protoc```,```protoc-gen-go```  
  2. 执行命令:
    ```go
    protoc --go_out=. *.proto
    ```
- 示例Demo


### 参考链接：
用Go构建gRPC  
https://medium.com/@shijuvar/building-high-performance-apis-in-go-using-grpc-and-protocol-buffers-2eda5b80771b  
拦截器：  
https://medium.com/@shijuvar/writing-grpc-interceptors-in-go-bf3e7671fe48