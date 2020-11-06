---
title: "URL 模块和 querystring 模块"
date: 2020-11-05T15:30:00+08:00
description: "Node.js 中的 url 模块和 querystring 模块"
draft: false
tags: [Node]
---

url 模块提供了解析 URL 的工具。```const url = require('url')`

url 模块提供了两套 [API](https://nodejs.org/dist/latest-v14.x/docs/api/url.html#url_url_strings_and_url_objects):  
- legacy API, Node.js 特有
- WHATWG URL Standard, 用于 web 浏览器  

```js
// WHATWG API
const myURL =
  new URL('https://user:pass@sub.example.com:8080/p/a/t/h?query=string#hash');
/**
> myURL
URL {
  href: 'https://user:pass@sub.example.com:8080/p/a/t/h?query=string#hash',
  origin: 'https://sub.example.com:8080',
  protocol: 'https:',
  username: 'user',
  password: 'pass',
  host: 'sub.example.com:8080',
  hostname: 'sub.example.com',
  port: '8080',
  pathname: '/p/a/t/h',
  search: '?query=string',
  searchParams: URLSearchParams { 'query' => 'string' },
  hash: '#hash'
}
*/

// Legacy API
const url = require('url');
const myURL =
  url.parse('https://user:pass@sub.example.com:8080/p/a/t/h?query=string#hash');
/**
> myURL
URL {
  href: 'https://user:pass@sub.example.com:8080/p/a/t/h?query=string#hash',
  origin: 'https://sub.example.com:8080',
  protocol: 'https:',
  username: 'user',
  password: 'pass',
  host: 'sub.example.com:8080',
  hostname: 'sub.example.com',
  port: '8080',
  pathname: '/p/a/t/h',
  search: '?query=string',
  searchParams: URLSearchParams { 'query' => 'string' },
  hash: '#hash'
}
*/
```

[获取 URL 中的 hash 值](https://nodejs.org/dist/latest-v14.x/docs/api/url.html#url_url_hash)：
```js
console.log(myURL.hash);
// Prints #bar

// 赋值给 hash 属性的值会被百分比编码（[percent-encoded](https://nodejs.org/dist/latest-v14.x/docs/api/url.html#whatwg-percent-encoding)），例如 " " 空格被编码成 "%20"
myURL.hash = 'baz';
console.log(myURL.href);
// Prints https://example.org/foo#baz
```
**注意**: http 模块在 `createServer((req, res) => {})` 方法中，request 事件监听器函数 req 参数的 [`url`](https://nodejs.org/dist/latest-v14.x/docs/api/http.html#http_message_url) 字段是无法接收到 hash 参数的，hash 部分会被丢弃，不会存在于报文的任何部分。hash 参数只用于客户端。

[url.searchParams](https://nodejs.org/dist/latest-v14.x/docs/api/url.html#url_url_searchparams): 获取表示 URL 中查询参数的 URLSearchParams 对象，这个属性只读，但是指向的 URLSearchParams 对象可以修改对应的 URL 实例。

## [URLSearchParams](https://nodejs.org/dist/latest-v14.x/docs/api/url.html#url_class_urlsearchparams)

URLSearchParams 挂载在全局对象 global 上。提供了读取和写入 URL query 参数的方法。与 [querystring](https://nodejs.org/dist/latest-v14.x/docs/api/querystring.html) 模块有类似的作用，但是后者更通用，比如后者允许自定义分隔符（& 和 =）。URLSearchParams API 只用于 URL query 参数。