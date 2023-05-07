---
title: "聊一聊Next.js的预渲染"
date: 2023-02-24
thumbnailImagePosition: right
categories:
  - FrontEnd
  - JavaScript
tags:
  - Next.js

metaAlignment: center
---

众所周知，`Next.js`具备在构建时预渲染、服务端渲染、路由预加载、智能打包、零配置等功能。提供多种渲染模式，可以用代理服务器将预渲染的页面提供给上游应用。

<!--more-->

![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/bg/brush.jpg)

### 前言

作为一个后端开发者在接触前端框架初期的时候，我通常面对新的技术名词之前，先会去了解它的定位。比如，`Reaction`是一个库，而`Next.js`是一个框架。

## 基础概念

### 库与框架

框架和库之间的技术差异在于一个称为控制反转的术语。当您使用库时，您负责应用程序流。您可以选择何时何地调用库。当您使用框架时，该框架负责流程。

本质上这意味着框架强加了一些您必须遵守的结构和约定。

然而，正因为如此，在`Next.js`中，一些事情，比如路由，是内置的，更容易实现。`Next.js`不需要安装单独的`NPM`包，如 Reaction-Router-Dom。

### 服务端渲染

> 服务器端呈现(SSR)是在服务器上呈现网页并将其传递给浏览器(客户端)，而不是在浏览器中呈现的过程。SSR 向客户端发送一个完全呈现的页面；客户端的`JavaScript`包接管并使 SPA 框架能够运行。

通常提到的服务器端呈现应用程序的优势之一是，当查看页面源代码时，在浏览器中公开了`HTML`标记(和其他项)，从而产生了很好的 SEO。在`Reaction`应用程序中，查看页面源代码几乎不会暴露搜索引擎爬行器可以使用的内容。

与 React 相比的另一个不同之处在于，Next.js 允许服务器呈现第一个页面加载。这使得它可能比 Reaction 应用程序更快，后者必须在访问页面之前加载整个应用程序。

使用 Next.js 的一般过程是，在服务器上执行您的 Reaction JSX 代码，并使用 HTML 创建一个页面并将其发送到用户的浏览器，完全呈现--HTML、CSS、数据和所有内容。

### 渲染流程

当使用代理服务器将预渲染的页面提供给上游应用时，通常需要以下步骤：

1. 在 Next.js 应用程序中实现 getStaticProps 或 getServerSideProps 方法，以获取数据并进行预渲染

```JavaScript
// pages/example.js

export async function getStaticProps() {
  const res = await fetch('https://example.com/api/data');
  const data = await res.json();

  return {
    props: {
      data,
    },
    revalidate: 60, // Enable static re-generation to update the cached page every minute
  };
}

function Example({ data }) {
  return (
    <div>
      <h1>{data.title}</h1>
      <p>{data.content}</p>
    </div>
  );
}

export default Example;
```

2. 构建和导出您的 Next.js 应用程序，生成静态 HTML 文件

```bash
npm run build && npm run export
```

3. 部署生成的静态 HTML 文件到独立的服务器上
4. 编写一个代理服务器，在接收来自上游应用的请求时，从独立服务器获取预渲染的 HTML，并将其返回给上游应用

```javascript
const express = require("express");
const httpProxy = require("http-proxy");

const app = express();
const proxy = httpProxy.createProxyServer();

app.use("/example", (req, res) => {
  const url = "http://your-static-server.com/example";
  proxy.web(req, res, { target: url });
});

app.listen(8080, () => {
  console.log("Server listening on port 8080");
});
```

这个例子中，代理服务器监听 8080 端口，在接收到/example 路径的请求时，会从http://your-static-server.com/example获取预渲染的HTML，并将其返回给上游应用。

需要注意的是，代理服务器需要设置正确的 CORS 头信息，以便上游应用可以正确地显示预渲染的内容。例如，在应用程序中，您可以通过设置以下头信息来允许所有来源的请求

```javascript
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  res.header(
    "Access-Control-Allow-Headers",
    "Origin, X-Requested-With, Content-Type, Accept"
  );
  next();
});
```

## 参考链接

- **building-smart-404-pages-in-nextjs**  
  https://articles.wesionary.team/building-smart-404-pages-in-nextjs-63fc5f10912b
- **什么是 Next.js 和服务器端渲染**  
  https://medium.com/javascript-in-plain-english/what-is-next-js-and-server-side-rendering-9e24ea21c144
