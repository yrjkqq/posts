---
title: "Node.js 的 child_process, cluster, worker_threads 模块"
date: 2019-01-14T16:41:00+08:00
description: "Node.js 中的 child_process, cluster, worker_threads 模块的异同和各自适用场景"
draft: false
tags: [Node]
---

Node 并非真正的单线程架构，有一些 I/O 线程由底层 libuv 处理。JavaScript 代码永远运行在 V8 上，是单线程的。

多线程中，每个线程都拥有自己独立的堆栈，这个堆栈需要占用一定的内存空间。由于一个 CPU 核心在一个时刻只能做一件事情，操作系统只能通过将 CPU 切分为时间片的方法，让线程可以较为均匀地使用 CPU 资源。但是操作系统在切换线程的同时也要切换线程的上下文，当线程数量过多时，事件会被消耗到上下文切换中。*Node 基于事件驱动，采用单线程避免了不必要的内存开销和上下文切换开销。*

Node 中所有处理都在单线程上进行，影响事件驱动服务模型性能的点在于 CPU 的计算能力，但它不受多进程或多线程中资源上限的影响，可伸缩性远比前两者更高。也就是说，CPU 计算能力可以通过增加 CPU 核心的数量来进行提高，进一步提高 Node 的性能。那么 Node 中如何利用多核 CPU 呢？

## 如何提高 CPU 的利用率？

解决单线程单进程堆多核使用不足的问题：理想状态下每个进程各自利用一个 CPU, 以此实现多核 CPU 的利用。Node 提供了 `child_process` 模块。

**Master-Worker 主从模式**：分布式架构中用于并行处理业务的模式。主进程不负责具体的业务处理，而是负责调度和管理工作进程，趋于稳定。工作进程负责具体的业务处理，需要关注其稳定性。示例代码如下：
```js
// master.js
import childProcess from "child_process";
import os from "os";
import { dirname } from "path";
import { fileURLToPath } from "url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// 获取操作系统的 cpu 的核心数
const cpus = os.cpus();
console.log(cpus.length); // 16
for (let i = 0; i < cpus.length; i++) {
  // 复制进程使每个 CPU 核心都利用上：fork() 复制的进程都是一个独立的进程，这个进程中有着独立而全新的 V8 实例。
  childProcess.fork(__dirname + "/worker.js");
}

// worker.js
#!/usr/bin/env node
import http from "http";
console.log("run worker.js");
http
  .createServer(function (req, res) {
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.end("Hello World!\n\r");
  })
  .listen(Math.round((1 + Math.random()) * 30000), "127.0.0.1");

// process.exit(0);
```

### 创建子进程

child_process 模块提供以下四个方法：使用方式查看 [文档](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_child_process_spawn_command_args_options)
- spawn()：返回的子进程对象具有 stdio 属性，可以用来获取子进程的输出。而 fork() 方法没有这个属性，无法以同样的方式获取子进程的输出。
- exec()
- execFile()
- [fork()](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_child_process_fork_modulepath_args_options): fork() 是 spawn() 的特例，用来创建 Node.js 进程。创建完返回的子进程会带有额外的内嵌的通信管道，用来在父子进程之间通信，具有 [send()](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_subprocess_send_message_sendhandle_options_callback) 方法。其余的 api 像是 spawn()，则不具有这个功能，返回的子进程对象没有 send() 方法，这些 api 可以用来创建非 Node.js 进程。

### 进程间通信

HTML5 提供了 WebWorkerAPI, WebWorker 运行创建工作线程并在后台运行，使得一些阻塞严重的计算不影响主线程上的 UI 渲染。线程之间通过 onmessage() 和 postMessage() 进行通信。类似地，在 Node 中使用 child_process 创建了子进程后会返回子进程对象，由 send() 方法实现主线程和子线程发送数据，message 事件实现收听子线程进程发来的数据。示例代码如下：
```js
// parent.js
import cp from "child_process";
import { getDirname } from "./util.js";

const __dirname = getDirname(import.meta.url);
const n = cp.fork(`${__dirname}/sub.js`);

n.on("message", function (m) {
  console.log("PARENT got message: ", m);
});
n.send({ hello: "world" });


// sub.js
process.on("message", function (m) {
  console.log("CHILD got message: ", m);
});

process.send({ foo: "bar" });

// util.js
import { dirname } from "path";
import { fileURLToPath } from "url";

export function getDirname(url) {
  const __filename = fileURLToPath(url);
  const __dirname = dirname(__filename);
  return __dirname;
}
```

#### 进程间通信原理

为了实现父子进程间的通信，父进程和子进程之间会创建 **IPC 通道（Inter-Process Communication, 进程间通信）**。Node 使用管道（pipe）实现 IPC 通道，管道具体实现由 libuv 提供，而在应用层只有 message 事件和 send() 方法。

**父进程在创建子进程之前，会创建 IPC 通道并监听它，然后才真正创建出子进程，并通过环境变量（NODE_CHANNEL_FD）告诉子进程这个 IPC 通道的文件描述符。子进程在启动的过程中，根据文件描述符去连接这个已存在的 IPC 通道，从而完成父子进程之间的连接。**

只有启动的子进程是 Node 进程时，子进程才根据环境变量去连接 IPC 通道。

### 传递句柄

多进程的 web 服务器架构中，通常是让每个进程监听不同的端口，其中主进程监听主端口（如 80），主进程对外接受所有的网络请求，再将这些请求分别代理到不同的端口的进程上。由于进程每接收到一个连接，将会用掉一个文件描述符，因此代理方案中客户端连接到代理进程，代理进程连接到工作进程的过程中需要用掉两个文件描述符。而操作系统的文件描述符是有限的，代理方案浪费掉一倍数量的文件描述符的做法影响了系统的扩展能力。

为了解决代理方案的问题，Node 引入了进程间发送句柄的功能。send() 方法的第二个参数就是可选句柄。句柄是一种可以用来标识资源的引用，它的内部包含了指向对象的文件描述符。

有了句柄后，代理方案中的主进程接收到 socket 请求后，将这个 socket 直接发送给工作进程，而不是重新与工作进程建立新的 socket 连接来转发数据。示例代码如下：
```js
// main.js
const cp = require("child_process");
const net = require("net");
// import cp from "child_process";
// import net from "net";

const subprocess = cp.fork("./handle-sub.js");

const server = net.createServer();
server.on("connection", function (socket) {
  socket.end("handled by parent\n");
});
server.listen(1337, function () {
  subprocess.send("server", server, (result) => {
    console.log(`parent send result: ${result}`);
  });
});

subprocess.on("message", function (m) {
  console.log(m);
});


// sub.js
#!/usr/bin/env node
process.on("message", (m, server) => {
  if (m === "server") {
    server.on("connection", (socket) => {
      console.log("subprocess got socket");
      socket.end("handled by child\n");
    });
  }
});
```
**注意**：如果使用 es6 的模块方案，上面的示例代码在使用 `curl localhost:1337` 如果请求到子进程，会卡住，curl 接收不到响应，子进程 socket connection 事件的监听器也打印不出日志。原因为止，改成 commonjs 的模块方式，一切正常。

主进程中的服务器在发送句柄后可以关闭服务，子进程可以在 http 层面来处理请求，示例代码如下：
```js
// parent.js
const cp = require("child_process");

const child1 = cp.fork("http-child.js");
const child2 = cp.fork("http-child.js");

const server = require("net").createServer();
server.listen(1337, function () {
  child1.send("server", server);
  child2.send("server", server);

  // 关掉
  server.close();
});

// child.js
const http = require("http");

const server = http.createServer(function (req, res) {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("handled by child, pid is " + process.pid + "\n");
});

process.on("message", function (m, tcp) {
  if (m === "server") {
    tcp.on("connection", function (socket) {
      server.emit("connection", socket);
    });
  }
});
```

---

由于 Node 执行在单线程上，一旦单线程上抛出的异常没有被捕获，将会引起整个进程崩溃。那么保证 Node 进程的健壮性和稳定性呢？使用 child_process 模块利用多核 cpu 创建多个进程后，每个工作进程仍然是单进程执行的，其稳定性如何保证？

## 提高进程的健壮性？

### 进程事件

除了 message 事件外，还有如下事件：  
- [error](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_event_error) 当子进程无法被复制创建、无法被杀死、无法发送消息时会触发该事件
- [exit](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_event_exit) 子进程退出时触发该事件，子进程如果是正常退出，这个事件的第一个参数为退出码，否则为 null. 如果进程是通过 kill() 方法被杀死的，会得到第二个参数，它表示杀死进程的信号
- [close](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_event_close) 在子进程的标准输入输出流中止时触发该事件，参数与 exit 相同
- [disconnect](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_event_disconnect) 在父进程或子进程中调用 disconnect() 方法时触发该事件，在调用该方法时将关闭监听 IPC 通道

[send()](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_subprocess_send_message_sendhandle_options_callback) 方法还可以接受第三个参数作为回调函数，该函数在 message 发送后但是可能在子进程接收到消息之前被调用。调用时传入一个参数，null 表示发送成功，Error 对象表示失败。

[kill()](https://nodejs.org/dist/latest-v14.x/docs/api/child_process.html#child_process_subprocess_kill_signal) 方法能给子进程发送消息。该方法并不能真正地将通过 IPC 通道相连的子进程杀死，它只是给子进程发送了一个系统信号。默认发送 SIGTERM 信号。
```js
// 子进程
child.kill([signal])

// 当前进程
process.kill(pid, [signal])
```

### 自动重启

通过进程事件，可以创造出管理进程的机制。比如在监听到子进程的 exit 事件后，主进程再重新一个工作进程来继续服务。示例代码如下：
```js
// reboot-master.js
const net = require("net");
const cp = require("child_process");
const os = require("os");

const server = net.createServer();
server.listen(1337);

const workers = {};
const createWorker = function () {
  const worker = cp.fork(__dirname + "/reboot-worker.js");
  // 退出时重新启动新的进程
  worker.on("exit", function () {
    console.log("Worker " + worker.pid + " exited.");
    delete workers[worker.pid];
    createWorker();
  });
  // 句柄转发
  worker.send("server", server);
  workers[worker.pid] = worker;
  console.log("Create worker, pid: " + worker.pid);
};

const cpus = os.cpus();
for (let i = 0; i < cpus.length; i++) {
  createWorker();
}

// 主进程退出时，退出所有工作进程
process.on("exit", function () {
  for (const pid in workers) {
    workers[pid].kill();
  }
});

// reboot-worker.js
process.on("message", function (m, server) {
  if (m === "server") {
    server.on("connection", function (socket) {
      socket.end(`handled by reboot-worker. pid: ${process.pid}\n`);
    });
  }
});
```
如果使用 `kill 子进程pid` 杀死一个子进程，那么会自动重新再创建一个子进程。如果直接使用 kill 命令杀死主进程（无论是 `kill -2 主进程pid` 模拟 ctrl+c 发送 SIGINT 信号，还是使用 `kill -9 主进程pid` 发送 SIGKILL 信号），那么子进程不会被杀死，需要用 `ps aux | grep reboot` 搜索出所有子进程后，使用 kill 命令一一杀死。但是直接在执行 `node reboot-master.js` 的命令行使用 ctrl+c 却可以停止所有子进程。主进程中接收到 SIGINT 信号主动退出，修改为以下代码：
```js
// 主进程退出时，退出所有工作进程
process.on("exit", function (code, signal) {
  console.log("exit", code, signal);
  for (const pid in workers) {
    workers[pid].kill();
  }
});

// 接收到 SIGINT 信号后(CTRL+C 或 kill -2 pid)，主进程退出，会触发 exit 事件，exit 事件中退出所有子进程
process.on("SIGINT", function (code, signal) {
  console.log("SIGINT", code, signal);
  process.exit(0);
});

// $ kill -2 pid
// SIGINT SIGINT 2
// exit 0 undefined
```

子进程中可能由于 BUG 抛出未捕获的错误，导致工作进程退出，此时也需要重新启动一个工作进程，保证整个集群中总是有工作进程为用户服务，示例代码如下：
```js
// reboot-worker.js
const url = require("url");
const http = require("http");

const server = http.createServer(function (req, res) {
  if (url.parse(req.url).pathname === "/error") {
    // 务必返回响应，否则连接会保持打开，无法断开，当前进程不会退出，新来的连接也会得不到响应
    res.writeHead(400);
    res.end("unknown url\n");
    throw new Error("unknown url");
  }
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("handled by chil, pid is " + process.pid + "\n");
});

let worker;

process.on("message", function (m, tcp) {
  if (m === "server") {
    worker = tcp;
    worker.on("connection", function (socket) {
      server.emit("connection", socket);
    });
  }
});

// https://nodejs.org/dist/latest-v14.x/docs/api/process.html#process_event_uncaughtexception
process.on("uncaughtException", function (error) {
  console.log("uncaughtException", error);
  // 停止接受新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1);
  });
});
```

以上代码问题在于需要等到所有连接断开后进程才退出，在极端的情况下，所有工作进程都停止接受新的连接，全处在等待退出的状态。比如上述代码中，如果没有返回 400 的响应，那么客户端的连接会一直保持打开，当前工作进程也无法接受新的请求。

但在等待进程全部退出重启的过程中，所有新来的请求可能存在没有工作进程微信用户服务的场景，这会丢掉大部分请求。在退出流程增加一个自杀信号，工作流程在得知退出时，向主进程发送一个自杀信号，然后才停止接受新的连接，当所有连接断开后才退出。主进程收到自杀信号后，立即创建新的工作进程服务。等待长连接断开可能需要较旧的时间，为此有必要为已有连接的断开设置一个超时时间。同时，异常退出记录日志有利于定位和追踪问题。完整代码如下：
```js
// master.js
const net = require("net");
const cp = require("child_process");
const os = require("os");

const server = net.createServer();
server.listen(1337);

const workers = {};
const createWorker = function () {
  const worker = cp.fork(__dirname + "/reboot-suicide-worker.js");
  worker.on("message", function (message) {
    if (message.act === "suicide") {
      createWorker();
    }
  });
  // 退出时重新启动新的进程
  worker.on("exit", function () {
    console.log("Worker " + worker.pid + " exited.");
    delete workers[worker.pid];
  });
  // 句柄转发
  worker.send("server", server);
  workers[worker.pid] = worker;
  console.log("Create worker, pid: " + worker.pid);
};

// const cpus = os.cpus();
for (let i = 0; i < 4; i++) {
  createWorker();
}

// 主进程退出时，退出所有工作进程
process.on("exit", function (code, signal) {
  console.log("exit", code, signal);
  for (const pid in workers) {
    workers[pid].kill();
  }
});

// 接收到 SIGINT 信号后(CTRL+C 或 kill -2 pid)，退出
process.on("SIGINT", function (code, signal) {
  console.log("SIGINT", code, signal);
  process.exit(0);
});

// sub.js
const url = require("url");
const http = require("http");
const winston = require("winston");

const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  defaultMeta: { service: "user-service" },
  transports: [
    //
    // - Write all logs with level `error` and below to `error.log`
    // - Write all logs with level `info` and below to `combined.log`
    //
    new winston.transports.File({ filename: "error.log", level: "error" }),
    new winston.transports.File({ filename: "combined.log" }),
  ],
});

const server = http.createServer(function (req, res) {
  if (url.parse(req.url).pathname === "/error") {
    // 务必返回响应，否则连接会保持打开，无法断开，当前进程不会退出，新来的连接也会得不到响应
    res.writeHead(400);
    res.end(`unknown url, handled by child, pid id ${process.pid}\n`);
    throw new Error("unknown url");
  }
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("handled by child, pid is " + process.pid + "\n");
});

let worker;

process.on("message", function (m, tcp) {
  if (m === "server") {
    worker = tcp;
    worker.on("connection", function (socket) {
      server.emit("connection", socket);
    });
  }
});

// https://nodejs.org/dist/latest-v14.x/docs/api/process.html#process_event_uncaughtexception
process.on("uncaughtException", function (error) {
  console.log("uncaughtException", error);
  // 记录日志
  logger.error(error.message);
  // 发送自杀信号
  process.send({ act: "suicide" });
  // 停止接受新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1);
  });
  // 5 秒后退出进程
  setTimeout(function () {
    // 超时退出
    console.log("timeout");
    process.exit(1);
  }, 5000);
});
```

为了避免程序编写错误导致子进程在启动中就发生错误，主进程进而无限重新创建子进程，应当设置规则，不应当反复重启。在单位时间内规定只能重启多少次，超过限制就触发 giveup 事件，放弃重启。代码如下：
```js
// master.js
const net = require("net");
const cp = require("child_process");
const os = require("os");

const server = net.createServer();
server.listen(1337);

// 重启次数
const limit = 10;
// 时间单位
const during = 60000;
let restart = [];
const isTooFrequency = function () {
  const time = Date.now();
  const length = restart.push(time);
  if (length > limit) {
    restart = restart.slice(length - limit, length);
  }
  return (
    restart.length >= limit && restart[restart.length - 1] - restart[0] < during
  );
};

const workers = {};
const createWorker = function () {
  const worker = cp.fork(__dirname + "/reboot-suicide-worker.js");
  worker.on("message", function (message) {
    if (message.act === "suicide") {
      if (isTooFrequency()) {
        // 触发 giveup 事件，不再重启
        process.emit("giveup", restart.length, during);
        return;
      }
      createWorker();
    }
  });
  // 退出时重新启动新的进程
  worker.on("exit", function () {
    console.log("Worker " + worker.pid + " exited.");
    delete workers[worker.pid];
  });
  // 句柄转发
  worker.send("server", server);
  workers[worker.pid] = worker;
  console.log("Create worker, pid: " + worker.pid);
};

const cpus = os.cpus();
for (let i = 0; i < cpus.length; i++) {
  createWorker();
}

process.on("giveup", function () {
  console.log("give up");
});

// 主进程退出时，退出所有工作进程
process.on("exit", function (code, signal) {
  console.log("exit", code, signal);
  for (const pid in workers) {
    workers[pid].kill();
  }
});

// 接收到 SIGINT 信号后(CTRL+C 或 kill -2 pid)，退出
process.on("SIGINT", function (code, signal) {
  console.log("SIGINT", code, signal);
  process.exit(0);
});


// sub.js
const url = require("url");
const http = require("http");
const winston = require("winston");

const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  defaultMeta: { service: "user-service" },
  transports: [
    //
    // - Write all logs with level `error` and below to `error.log`
    // - Write all logs with level `info` and below to `combined.log`
    //
    new winston.transports.File({ filename: "error.log", level: "error" }),
    new winston.transports.File({ filename: "combined.log" }),
  ],
});

const server = http.createServer(function (req, res) {
  if (url.parse(req.url).pathname === "/error") {
    // 务必返回响应，否则连接会保持打开，无法断开，当前进程不会退出，新来的连接也会得不到响应
    res.writeHead(400);
    res.end(`unknown url, handled by child, pid id ${process.pid}\n`);
    throw new Error("unknown url");
  }
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("handled by child, pid is " + process.pid + "\n");
});

let worker;

process.on("message", function (m, tcp) {
  if (m === "server") {
    worker = tcp;
    worker.on("connection", function (socket) {
      server.emit("connection", socket);
    });
    // 模拟在启动中抛出异常
    throw new Error("error in startup");
  }
});

// https://nodejs.org/dist/latest-v14.x/docs/api/process.html#process_event_uncaughtexception
process.on("uncaughtException", function (error) {
  console.log("uncaughtException", error);
  // 记录日志
  logger.error(error.message);
  // 发送自杀信号
  process.send({ act: "suicide" });
  // 停止接受新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1);
  });
  // 5 秒后退出进程
  setTimeout(function () {
    // 超时退出
    console.log("timeout");
    process.exit(1);
  }, 5000);
});

```

### 负载均衡

Node 默认提供的机制是采用操作系统的抢占式策略，各个进程根据自己的繁忙程度来进行抢占。Node 进程的繁忙由 CPU, I/O 两个部分构成，影响抢占的是 CPU 的繁忙度。如果 I/O 繁忙而 CPU 空闲则会造成某个进程能抢到较多请求，导致负载不均衡。

为此 Node 提供了新的策略 **Round-Robin 轮叫调度**。主进程接受连接，将其依次分发给工作进程。策略是在 N 个工作进程中，每次选择第 i = (i + 1) mod n 个进程来发送连接。

### 状态共享

Node 不允许在多个进程之间共享数据。实际业务中需要共享一些数据，可以采用以下几种方案：

1. 第三方数据存储
   将数据存放到数据库、磁盘文件、缓存服务（如 redis）中，所有工作进程在启动时将其读取到内存中。状态发生改变，需要通知到各个子进程。  
   **定时轮询**：轮询时间的设置是关键。
   **主动通知**：当数据发生更新时，主动通知子进程。专门用来发送通知和查询状态是否更改的进程叫做通知进程。这个进程只进行轮询和通知，不处理任何业务逻辑。推送机制如果按照进程间信号传递，在跨多台服务器时会失效，可以采用 tcp 或 udp 的方案。进程在启动时从通知服务处除了读取第一次数据外，还将进程信息注册到通知服务处。通知进程通过轮询发现数据更新后，根据注册信息，将更新后的数据发送给工作进程。

## cluster 模块

Node.js v0.8 版本引入 cluster 模块，用以解决多核 CPU 的利用率问题同时也提供了较为完善的 API 用于处理进程的健壮性问题。

```js
const cluster = require("cluster");
const http = require("http");
const os = require("os");

const numCpus = os.cpus().length;

if (cluster.isMaster) {
  console.log(`Master ${process.pid} is running`);

  // Fork workers
  for (let i = 0; i < numCpus; i++) {
    cluster.fork();
  }

  cluster.on("exit", function (worker, code, signal) {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  // Workers can share ant TCP connection
  // In this case it is a HTTP server
  http
    .createServer(function (req, res) {
      res.writeHead(200);
      res.end("hello world\n");
    })
    .listen(8000);
  console.log(`Worker ${process.pid} started`);
}
```

cluster 模块就是 child_process 和 net 模块的组合应用。cluster 启动时，内部会启动 tcp 服务器，在 cluster.fork() 子进程后，将这个 tcp 服务器端 socket 的文件描述符发送给工作进程。如果进程是通过 cluster.fork() 复制出来的，那么它的环境变量里就存在 NODE_UNIQUE_ID, 如果工作进程中存在 listen 侦听网络端口的调用，它将拿到该文件描述符，通过 SO_REUSEADDR 端口重用，从而实现多个子进程共享端口。对于普通方式启动的进程，则不存在文件描述传递共享等事情。

## worker_threads 模块

worker_thread 模块使得并行执行 JavaScript 线程成为可能。workers(threads) 对于 CPU 密集型的操作很有用，对于 I/O 密集型任务，使用 Node.js 内置的 I/O 操作更高效。 

与 child_process 或 cluster 模块不同的是，worker_threads 能够共享内存。通过传递 ArrayBuffer 实例或共享 SharedArrayBuffer 实例达到这一目标。

```js
const {
  Worker,
  isMainThread,
  parentPort,
  workerData,
} = require("worker_threads");

if (isMainThread) {
  module.exports = function parseJSAsync(script) {
    return new Promise((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: script,
      });
      worker.on("message", resolve);
      worker.on("error", reject);
      worker.on("exit", (code) => {
        if (code != 0) {
          reject(new Error(`Worker stopped with exit code ${code}`));
        }
      });
    });
  };
} else {
  const { parse } = require("some-js-parse-library");
  const script = workerData;
  parentPort.postMessage(parse(script));
}
```

worker_threads 模块：https://nodejs.org/dist/latest-v14.x/docs/api/worker_threads.html