---
title: "《深入浅出 Node.js》学习笔记"
date: 2019-01-02T16:41:00+08:00
description: "记录看 《深入浅出 Node.js》一书过程中遇到的知识点和问题，方便日后回顾。"
draft: false
tags: [Node]
---

## 第 3 章 异步 I/O

异步 I/O 主要关注点在**用户体验**和**资源分配**。

用户体验方面，异步方式在响应请求方面显著降低了同步请求的等待时间，由 M+N 降低为 max(M+N)。

Node 利用单线程，解决了多线程编程模型中死锁、状态同步等问题，结合异步 I/O 解决了单线程串行依次执行编程模型因阻塞 I/O 导致资源得不到更优应用的问题。

### 操作系统对异步 I/O 的支持

非阻塞 I/O 在调用后会立即返回，但是由于完整的 I/O 并没有完成，立即返回的并不是业务层期望的数据，而仅仅时调用状态。为了获取完整的数据，应用程序需要重复调用 I/O 操作即**轮询**来确认是否完成。

现有的轮询技术主要有：read select poll epoll kqueue

单线程中实现异步 I/O 总会有各种缺陷，但是利用**多线程**，通过让部分线程进行阻塞 I/O 或者非阻塞 I/O 加轮询技术来完成数据获取，让一个线程进行计算处理，通过线程之间的通信将 I/O 得到的数据进行传递，即可**模拟**出异步 I/O.

Node 在 linux 下通过自行实现线程池，在 windows 下使用 IOCP(内核管理的线程池)，并且提供 libuv 对平台差异性进行抽象封装，来实现异步 I/O.

### Node 如何实现异步 I/O

完成整个异步 I/O 环节的有事件循环、观察者模式、请求对象、I/O 线程池。

事件循环：Node 自身的执行模型。在进程启动时，Node 便会创建一个类似于 while(true) 的循环，每执行一次循环体的过程我们成为 Tick。每个 Tick 的过程就是查看是否有事件等待处理，如果有则取出事件和对应的回调函数。如果存在关联的回调函数，就执行。然后进入下个循环，直到不再有事件处理。

每个事件循环有一个或多个观察者，而判断是否有事件要处理的过程则是向这些观察者询问是否有要处理的事件。

生产者/消费者模型：异步 I/O、网络请求等则是事件的生产者，为 Node 提供不同类型的事件，这些时间被传递到对应的观察者那里，事件循环则从观察者那里取出事件并处理。

![整个异步 I/O 的执行流程](/posts/img/Snipaste_2020-10-30_08-02-28.png)

在 Node 中，js 是在单线程执行，而 Node 自身是多线程运行的。

### 非 I/O 的异步 API

`process.nextTick(() => {})`（在下一轮 Tick 时执行，时间复杂度 O(1)） 比 `setTimeout(() => {},0)`（内部会用到红黑树，对数时间复杂度） 更高效。

`setImmediate()` 与 `process.nextTick()` 方法类似。同时使用时 nextTick 方法会优先执行，原因在于：事件循环对观察者的检查是有先后顺序的，`process.nextTick()` 属于 idle 观察者，`setImmediate()` 属于 check 观察者，在每一轮循环检查中，idle 观察者先于 I/O 观察者，I/O 观察者先于 check 观察者。

实现上，`process.nextTick()` 回调函数保存在一个数组中，`setImmediate()` 的回调函数保存在链表中，而在行为上，`process.nextTick()` 在每轮循环中会将数组中的回调函数全部执行，而 `setImmediate()` 在每轮循环中执行链表中的一个回调函数。

### 高性能与网络服务器

Node 使用异步 I/O 处理网络套接字，操作系统端口上侦听到的网络请求会发送给 I/O 观察者形成事件，而事件循环则不断处理这些网络 I/O 事件，如果 JS 有传入回调函数，这些事件最终传递到业务层进行处理。

经典服务器模型（如 Apache）采用每线程/每请求的方式，为每个请求启动一个线程来处理。Node 之所以高效，是由于其通过事件驱动的方式处理请求，不用为每一个请求创建额外的对应线程，可以节省创建线程和销毁线程的开销，同时操作系统在调度任务时由于线程较少，上下文切换代价很低。

## 第 4 章 异步编程

异步编程在流程控制中，业务表达并不太适合自然语言的线性思维习惯。

### 函数式编程

高阶函数：把函数作为参数，或是将函数作为返回值的函数。

偏函数用法：指创建一个调用另外一部分——参数或变量已经预置的函数——的函数的用法。
```js
var _ = {};
_.after = function (times, func) {
  if (times <= 0) return func();
  return function () {
    if (--times < 1) {
      return func.apply(this, arguments);
    }
  };
};

var afterFunc = _.after(3, (i) => console.log(i));
afterFunc(1);
afterFunc(2);
afterFunc(3); // 此时开始执行
afterFunc(4);
```

### 异步编程的优势与难点

优势：擅长处理 I/O 密集型任务，只要计算不影响异步 I/O 的调度即可。

难点1：异常处理

Node 处理异常的约定：将异常作为回调函数的第一个实参返回，如果为空值，则表明异步调用没有抛出异常。编写异步方法遵循以下原则：  
- 必须执行调用者传入的回调函数
- 正确传递异常供调用者判断

难点2：回调函数嵌套过深

难点3：阻塞代码  
合理使用 setTimeout()

难点4：多线程编程
Node 借鉴浏览器 Web Workers 将 JS 执行和 UI 渲染分离更好利用多核 CPU 的模式，提出基础 API child_process 和 cluster 模块。

难点5：异步转同步

### 异步编程解决方案

#### 事件发布/订阅模式：events 模块

典型应用场景：通过事件发布/订阅模式进行组件封装，将不变的部分封装在组件内部，将容易变化、需要自定义的部分通过事件暴露给外部处理，这是一种典型的逻辑分离方式。

使用 events 模块：

1. 继承 events 模块
2. 利用事件队列解决雪崩问题  
雪崩问题：在高访问量、大并发量的情况下缓存失效的场景，此时大量请求涌入数据库中，数据库无法同时承受如此大的查询请求，进而影响到网站整体的响应速度。  
once() 方法执行一次就会将监视器移除  
```js
var events = require("events");

var proxy = new events.EventEmitter();
var status = "ready";
var select = function (callback) {
  // 多次调用 select 会给 proxy 注册多个监听器
  proxy.once("selected", callback);
  if (status === "ready") {
    status = "pending";
    db.select("SQL", function (results) {
      proxy.emit("selected", results);
      status = "ready";
    });
  }
};
```
3. 多异步之间的协作方案  
异步编程中，也会出现事件与侦听器的关系是多对一的情况，也就是说一个业务逻辑可能依赖两个通过回调或事件传递的结果，此时可能导致难点2：回调地狱。  
```js
var after = function (times, callback) {
  var count = 0,
    results = {};
  return function (key, value) {
    results[key] = value;
    count++;
    if (count === times) {
      callback(result);
    }
  };
};
// 利用偏函数完成多对一的收敛
var done = after(times, render);

var emitter = new events.Emitter();
// 利用事件订阅/发布模式完成一对多的发散
emitter.on("done", done);
emitter.on("done", other);
fs.readFile(template_path, "utf8", function (err, template) {
  emitter.emit("done", "template", template);
});
db.query(sql, function (err, data) {
  emitter.emit("done", "data", data);
});
i10n.get(function (err, resources) {
  emitter.emit("done", "resources", resources);
});
```

*本章节剩余部分已经过时，移步 [了解 JavaScript Promise](http://nodejs.cn/learn/understanding-javascript-promises) 和 [《ECMAScript6 入门》Promise 对象](https://es6.ruanyifeng.com/#docs/promise) 学习基于 Promise 或 Async/Await 的异步编程解决方案，以及 [前端面试题 1.4 Promise](https://juejin.im/post/6844904116339261447#heading-44) 练习相关题型*

---

#### Promise  

ES2015 引入 promise 并进行了标准化。ES2017 拥有了 async/await 语法。

Promisifying 技术：使用经典的 JavaScript 函数来接受回调并使其返回 promise:  
```js
const fs = require('fs')

const getFile = (fileName) => {
  return new Promise((resolve, reject) => {
    fs.readFile(fileName, (err, data) => {
      if (err) {
        reject (err)  // 调用 `reject` 会导致 promise 失败，无论是否传入错误作为参数，
        return        // 且不再进行下去。
      }
      resolve(data)
    })
  })
}

getFile('/etc/passwd')
.then(data => console.log(data))
.catch(err => console.error(err))
```  
最新版本的 Node.js 中，使用 [util 模块](http://nodejs.cn/api/util.html#util_util_promisify_original)的 promisifying 函数进行转换。

#### async/await

在任何函数之前加上 async 关键字意味着该函数会返回 promise。async/await 相比于 promise, 代码更简洁，也易于调试。

*剩余内容参见[promise](../04-promise)*