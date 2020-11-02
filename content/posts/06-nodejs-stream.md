---
title: "Node.js 中的 stream"
date: 2018-09-08T19:41:00+08:00
description: "流（stream）是 Node.js 中处理流式数据的抽象接口。"
draft: false
tags: [Node]
---

stream 模块用于构建实现了流接口的对象。Node.js 提供了多种流对象。流可以是可读的、可写的、或者可读可写的。所有的流都是 EventEmitter 的实例。

流的类型：  
- Writable 可写入数据的流，如 `fs.createWriteStream()`
- Readable 可读取数据的流，如 `fs.createReadStream()`
- Duplex 可读又可写的流，如 `net.Socket`
- Transform 在读写过程中可以修改或转换数据的 Duplex 流，如 `zlib.createDeflate()`

Node.js 创建的流都是运行在字符串和 Buffer (或 Uint8Array) 上。流的实现也可以使用其他类型的 JavaScript 值（除了 null）。

使用 stream 实现的 http 服务器:  
```js
import http from "http";

const server = http.createServer(function (req, res) {
  // req 是一个 http.IncomingMessage 实例，是可读流
  // res 是一个 http.ServerResponse 实例，是可写流

  let body = "";
  // 接收数据为 utf8 字符串
  // 如果没有设置字符编码，则会接收到 Buffer 对象
  req.setEncoding("utf8");

  // 如果添加了监听器，则可读流会触发 'data' 事件
  req.on("data", (chunk) => {
    body += chunk;
  });

  // 'end' 事件表明整个请求体已被接受
  req.on("end", () => {
    try {
      const data = JSON.parse(body);
      // 响应信息给用户
      res.write(typeof data);
      res.end();
    } catch (err) {
      res.statusCode = 400;
      return res.end(`错误：${err.message}`);
    }
  });
});
server.listen(1337);

// $ curl localhost:1337 -d "{}"
// object
// $ curl localhost:1337 -d "\"foo\""
// string
// $ curl localhost:1337 -d "not json"
// 错误: Unexpected token o in JSON at position 1

```