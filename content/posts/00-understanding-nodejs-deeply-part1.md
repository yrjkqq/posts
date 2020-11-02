---
title: "《深入浅出 Node.js》学习笔记 Part1"
date: 2019-01-02T16:41:00+08:00
description: "异步 I/O, 异步编程，内存控制"
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

## 第 5 章 内存控制

### V8 的内存限制和垃圾回收机制

Node 使用 V8 做为 Node 的 JavaScript 脚本引擎。

由于 V8 的内存管理机制，Node 中通过 JavaScript 只能使用部分内存 (64 位系统下约为 1.4GB, 32 位系统下约为 0.7GB), 导致 Node 无法直接操作大内存对象。

Node 中使用的 JavaScript 对象基本上都是通过 V8 自己的方式进行分配和管理的。V8 通过堆来为 JavaScript 对象分配内存。V8 中堆内存的大小是固定的，可以在启动时通过参数设置，无法根据使用情况自动扩充。

V8 为啥要限制堆的大小？  
- V8 最初为浏览器设计，不太可能遇到需要大量内存的场景
- V8 的垃圾回收运行中会引起 JavaScript 线程暂停执行，内存越大所需时间越长，导致应用的性能和响应下降


#### V8 的垃圾回收机制

V8 主要的垃圾回收算法：  
分布式垃圾回收机制：按照对象的存活时间将内存的垃圾回收进行不同的分代，然后对不同分代的内存施以更高效的算法。

V8 将内存分为新生代（存活时间较短的对象）和老生代（存活时间较长或常驻内存的对象）。

新生代的对象主要通过 Scavenge 算法进行来垃圾回收。该算法使用复制的方式实现垃圾回收。将堆内存一分为二，每一部分空间称为 semispace. 在垃圾回收的过程中，就是通过将存活对象在两个 semispace 空间之间进行复制。

当一个对象经过多次复制依然存活时，就会被认为是生命周期较长的对象，这种对象随后会被转移到老生代中，这个过程称为晋升。

老生代的对象主要通过 Mark-Sweep(标记清除) 和 Mark-Compact(标记整理) 相结合的方式进行垃圾回收。

Mark-Sweep 分为标记和清除两个阶段。标记阶段遍历堆中的所有对象，并标记活着对象，在随后的清除阶段中，只清除没有被标记的对象。

对比：Scavenge 中只复制活着的对象，而 Mark-Sweep 只清理死亡对象。或对象在新生代中只占较少部分，死对象在老生代中只占较少部分。

Mark-Sweep 的问题在于：在进行一次标记清除回收后，内存空间会出现不连续的状态，这种内存碎片在出现需要分配一个大对象时，连续内存空间不足，会触发不必要的垃圾回收。因此出现了 Mark-Compact，在对象被标记为死亡后，整理过程中，将活着的对象往一段移动，移动完成后，直接清理边界外的内存。

Mark-Compact 由于需要移动对象，执行速度不快。实际上，V8 主要使用 Mark-Sweep, 在空间不足以堆新生代晋升过来的对象进行分配时才使用 Mark-Compact.

垃圾回收阶段会将应用逻辑暂停下来，完成后再恢复执行应用逻辑，这种行为被称为“全停顿”(stop-the-world). 老生代堆内存配置的较大，且存活对象较多，全堆垃圾回收的标记、清除、整理等造成的停顿比较明显。为了降低停顿时间，V8 将一口气停顿完成的动作改为增量标记（incremental marking）.

查看垃圾回收日志：  
```js
/**
 *  node --trace_gc -e "var a = [];for(var i = 0; i < 1000000;i++){a.push(new Array(100))}" > gc.log
 *
 * --trace_gc 会从标准输出中打印垃圾回收的日志信息
 */

 /**
  * 在 Node 启动时使用 --prof 参数，可以得到 V8 执行时的性能分析数据，其中包含了垃圾回收执行时占用的时间
  * node --prof gc.js
  */
for (var i = 0; i < 1000000; i++) {
  var a = {};
}
```

### 高效使用内存

#### 作用域

调用函数时会创建对应的作用域，执行结束后销毁作用域。作用域中声明的局部变量也随着作用域的销毁而销毁。只被局部变量引用的对象存活周期较短，将会被分配在新生代中，下次垃圾回收时被释放。

全局作用域需要直到进程退出后才能释放，其中的全局变量引用的对象常驻在老生代内存中。

变量的主动释放：通过 delete 操作符来删除引用关系，或是将变量重新赋值，让旧的对象脱离引用关系。

闭包：实现外部作用域访问内部作用域中变量的方法叫做闭包。  
```js
var foo = function() {
  var bar = function() {
    var local = "内部变量";
    return function() {
      return local;
    }
  }
  var baz = bar()
  console.log(baz())
}
```  
闭包的问题在于一旦有变量引用这个中间函数，这个中间函数将不会释放，同时也会使原始的作用域不会得到释放，作用域中产生的内存占用也不会得到释放。

#### 内存指标

没有被回收的对象不断增长会导致内存占用无线增长，达到 V8 的内存限制后就会得到内存溢出错误，进而导致进程退出。

查看进程的内存占用：
```js
> process.memoryUsage()
{
  rss: 24821760,
  heapTotal: 5046272,
  heapUsed: 3322952,
  external: 1633208,
  arrayBuffers: 386230
}
```
rss 是 resident set size 的缩写，即进程的常驻内存部分。heapTotal 和 headUsed 对应的是 V8 的堆内存信息。heapTotal 是堆中总共申请的内存量，headUsed 表示目前我们堆中使用的内存量。单位都是字节。

本机上当 heapTotal 达到 headTotal 4195.41MB 是会出现内存溢出。  
```js
<--- Last few GCs --->

[22132:000002AE5F551500]     1973 ms: Scavenge 3842.5 (3875.9) -> 3842.5 (3875.9) MB4.3 / 0.0 ms  (average mu = 1.000, current mu = 1.000) allocation failure
[22132:000002AE5F551500]     2089 ms: Mark-sweep 4002.5 (4035.9) -> 4001.9 (4035.4)  49.8 / 0.0 ms  (+ 62.7 ms in 23 steps since start of marking, biggest step 5.5 ms, ltime since start of marking 1901 ms) (average mu = 1.000, current mu = 1.000) al   

<--- JS stacktrace --->

FATAL ERROR: MarkCompactCollector: young object promotion failed Allocation failed -vaScript heap out of memory
 1: 00007FF77B201DDF napi_wrap+109135
 2: 00007FF77B1A6D06 v8::internal::OrderedHashTable<v8::internal::OrderedHashSet,1>:mberOfElementsOffset+3335
```

查看系统的内存占用：  
os 模块的 `totalmem()` 和 `freemem()` 两个方法用于查看操作系统的内存占用情况，分别返回系统的总内存和闲置内存，以字节为单位。
```js
> os.totalmem()/1024/1024/1024
15.903568267822266
> os.freemem()/1024/1024/1024
5.802955627441406
> process.memoryUsage().rss/1024/1024/1024
0.0236968994140625
```

堆中的内存用量总是小于进程的常驻内存用量，意味着 Node 中的内存并非都是通过 V8 进行分配的，将那些不是通过 V8 分配的内存成为**堆外内存**。

Buffer 对象不同于其他对象，不经过 V8 的内存分配机制，不会有堆内存的大小限制。由于 Node 并不同于浏览器的应用场景，浏览器中 JavaScript 直接处理字符串即可满足大部分业务场景，而 Node 则需要网络流和文件 I/O 流，操作字符串远不能满足传输的性能需求。

#### 内存泄漏

内存泄漏的实质：应当回收的对象出现意外而没有被回收，变成了常驻在老生代中的对象。

内存泄露的原因：  
1. 缓存  
   缓存中的对象常驻在老生代中，不会被垃圾回收。  
   使用对象做为缓存时，需要加入策略来限制缓存的无限增长。参考 LRU 算法的实现。  
   模块机制：为了加速模块引入，所有模块都会通过编译执行，然后被缓存。而且通过 exports 导出的函数，可以访问模块中的私有变量，这样文件模块在文件编译执行后形成的作用域因为模块缓存的原因，不会被释放。模块是常驻在老生代中的。设计模块时需要提供接口，以便调用者主动释放内存。

   解决方案：内存做为缓存，除了需要限制大小外，也无法在进程之间共享。通过使用进程外缓存，Node 可以减少常驻内存的对象的数量，让垃圾回收更高效，同时进程间也可以共享缓存。可以使用 Redis.  
2. 队列消费不及时  
   消费者-生产者模型中，消费速度低于生产速度，将会形成堆积。比如往数据库写入数据，突然间写入大量数据将会造成阻塞，JavaScript 中相关的作用域也不会得到释放，内存泄露。

   解决方案：1. 换用消费速度更高的技术。2. 监控队列的长度。3. 为异步调用加入超时机制，为消费速度设置下限值。
3. 作用域未释放
   代码中闭包使用太多，会导致作用域过长，无法及时释放变量。

#### 内存泄漏排查

工具：  
- v8-profiler 四年没有维护
- node-heapdump 对 V8 堆内存抓取快照，用于事后分析
- [@airbnb/node-memwatch](https://www.npmjs.com/package/@airbnb/node-memwatch)

#### 大内存应用

使用 stream 模块用于操作大文件。