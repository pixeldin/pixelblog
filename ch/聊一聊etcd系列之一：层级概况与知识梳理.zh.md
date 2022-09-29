---
title: "聊一聊etcd系列之一：层级概况与知识梳理"
date: 2022-05-17
thumbnailImagePosition: left
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/s0-%E8%84%91%E5%9B%BE.png
categories:
- Go
- 后端
tags:
- raft
- etcd
- 分布式系统

metaAlignment: center
---

etcd在微服务领域使用场景十分普遍，经常用于作为一个注册中心，提供服务注册与发现。本系列将聊一聊etcd解决了什么问题、它是如何解决的角度来逐步分析。
<!--more-->

![etcd-cover-tree](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/etcd-tree.png)

## 前言
etcd在微服务领域使用场景十分普遍，经常用于作为一个注册中心，提供服务注册与发现常见的比如K8S框架就以etcd作为基础组件进行内部交互。近期在梳理etcd的一些内部结构与依赖组件，发现其背后有广泛的技术栈，这里分享总结一些技术细节作为沉淀。
## 它解决了什么问题
etcd的定位是通用的一致性KV存储，但在面向服务注册与发现的应用场景中，大概有这几个使用方式：
1. 注册/配置中心 KV存储
2. 服务发现
3. 分布式锁

## 它是如何解决的
### 分层设计
分层设计是软件开发的基本思路，etcd也采取模块分工，将各个内部组件分开实现，做到各司其职，从API层到逻辑层，再到底层文件存储，大体拆分为三大模块，本系列将依据这个思路，逐章拆解。  
今天尝试先从整体视角梳理大部分的技术栈，关于细节实现将在后续源码分析深入介绍。**这个系列将拆分为下面几个大模块进行分析，如果感兴趣的可以关注一下~**
### 一致性协议
提到分布式存储，分布式协议自然是离不开的，etcd底层依靠**raft**一致性算法保障一致性，采用**选举策略**定期更新各个节点当前状态。
### 通信方式
etcd节点间通信方式是经过更新迭代的(v2->v3)，v2使用HTTP/1.X协议，基于RESTful API通信；v3在兼容v2的基础上增加了HTTP/2.x grpc协议的支持，在多路复用的场景提升了传输效率，基于[Stream API](https://etcd.io/docs/v3.5/learning/design-client/)长连接进行活性检测。  
![image](https://etcd.io/docs/v3.5/learning/img/client-balancer-figure-06.png)
此外，etcd还暴露了友好API屏蔽底层的复杂的应用程序逻辑，提供多台物理机的一个逻辑集群视图。

### 数据存储
关于etcd数据的底层存储，主要依赖于预写日志(WAL)以及BoltDB存储引擎来实现，所有数据在提交之前会保障写入WAL做数据持久化，BoltDB的细节实现将会在后续系列展开细说，大概用到这几个技术点：
- **B+树**、KV数据库，**“读多写少”**
- 结合**COW**无锁读写并发
- **mmap**(内存映射磁盘技术)，零拷贝  

其中etcd花了大量的篇幅来做节点的多版本控制(**MVCC**)，依赖的是底层B+树和内存B树结构的索引映射，它们之间是如何关联的将在后面另起篇幅展开聊聊。

## 脉络图
关于etcd整个技术栈其实可以纵横扩展，这里梳理了一张脉络图，这个系列会结合这个脑图进行开展，大家也可以结合自己感兴趣的点进行深入接触。
![image](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/etcd/raft/s0-%E8%84%91%E5%9B%BE.png)

## 参考链接
- etcd文档  
https://etcd.io/docs/v3.5/learning/
- Deep Dive: etcd  
https://www.youtube.com/watch?v=DrtdrdwDpZE
- raft pdf  
https://raft.github.io/raft.pdf