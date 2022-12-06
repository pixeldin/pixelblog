---
title: "OpenResty Notes"
date: 2022-11-13
thumbnailImagePosition: right
<!-- thumbnailImage: /img/go-context.jpg -->
thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/bird.png
categories:
- Lua
- gateway
tags:
- Nginx
- OpenResty
- LuaJIT

metaAlignment: center
---

Lua作为一种胶水语言，具备动态类型，通过整合已有的高级组件构建新的应用。
<!--more-->
![openresty-home](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/Lua/openresty/bird-home.png)

# ❓特性是什么
💡 Nginx网关+插件化管理
- 特性一: 同步非阻塞
- 特性二: 动态热更新
- 特性三: 测试用例驱动完善

## 前言
从Nginx出发，再到插件定制化，作为一个国产API网关的沉淀，OpenResty开始逐渐摆脱NGINX的影子，形成自己的生态体系，在API网关、软WAF等领域被广泛使用。

## 🔨如何实现

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

### Lua & LuaJIT
Lua: 虚拟机 编译字节码→机器码

LuaJIT: 在上面基础上增加热代码检测, 对于频繁调用代码开启缓冲, 自己转成机器码

> **所谓 LuaJIT 的性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式退到 Lua 解释器的解释执行模式**
>

## 🔗参考资料
[https://github.com/moonbingbing/openresty-best-practices](https://github.com/moonbingbing/openresty-best-practices)