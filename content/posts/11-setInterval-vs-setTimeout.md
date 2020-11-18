---
title: "setTimeout 和 setInterval"
date: 2020-11-19T01:50:00+08:00
description: "比较 setTimeout 和 setInterval 的使用差异和各自使用场景"
draft: false
tags: [JavaScript]
---

JavaScript 如何模拟 sleep 的函数效果？

在 es5 中可以使用 while(){} 循环，循环条件为当前时间戳与循环开始前时间戳的的差值，如果小于 sleep 的时间间隔，则继续循环，以此模拟。

而在 es6 中可以借助 async await 的语法更优雅且高效的实现这一效果，代码参考 [stack overflow](https://stackoverflow.com/a/39914235/12796820) 如下：
```js
function sleep(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

async function wait() {
  await sleep(2000);
  console.log("2 seconds later")
}
```

有了 sleep 函数之后可以验证 setTimeout 和 setInterval 的具体执行效果。查找资料 [Scheduling: setTimeout and setInterval](https://javascript.info/settimeout-setinterval#nested-settimeout) 发现：在使用 setInterval 的过程中，会发现前后两次函数的执行时间间隔（上一次执行完成到下一次调用开始）并不是精确的等于指定的延迟时间，是因为 setInterval 是由 JavaScript 引擎进行调度的，每隔指定的延迟时间就会执行回调函数。引擎会等待回调函数执行完成，然后检查调度器如果时间到了指定的延迟时间，则立即执行下一次调用。而回调函数本身执行需要一定时间，这个时间是计算在指定的延迟时间内的，所以前一次调用结束到下一次调用开始的时间间隔是小于指定的延迟时间的。但是，如果函数执行的时间超过了延迟时间，那么下一次调用会立即开始，这种情况下实际的延迟时间会大于执行的延迟时间。

同时在 setInterval 使用前述的 sleep 函数是没有效果的，代码如下。会依次打印 2 2 5 2 5 ..., 其中一开始连续打印两个 2 后才开始打印 5，由此可见 sleep 实际上还是在上一次调度中，结束后打印 5，而此时第三次调度已经开始了。setInterval 的调度并没有等待 sleep 的完成。推测是由于引擎的调度形式有关，会严格按照指定延迟进行调度，而不会将函数执行时间计算在调度内（前文资料中的说法不准确，下一次调用不会等待上一次回调完成，调度器检测到延迟时间已到则会立即开始下一次），所以如果函数本身执行时间大于延迟时间，那么就会出现上一次调用没完成，下一次已经开始了。如果是在发送请求的场景中，可能会造成服务器无法及时响应而请求还在不断累积的现象，超负载。
```js
let last = Date.now();

function sleep(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

setInterval(async () => {
  const now = Date.now();
  console.log("2");
  last = now;
  await sleep(3000);
  console.log("5");
}, 2000);

// 2 2 5 2 5 
```

为了避免出现函数执行时间超过延迟时间导致调用积压，需要精确的控制两次函数的执行间隔，上一次调用完成后必须等待指定时间才能开始下一次延迟。递归式的调用 setTimeout 可以实现。代码如下：
```js
let last = Date.now();

function sleep(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

setTimeout(async function wait() {
  const now = Date.now();
  console.log("2");
  last = now;
  await sleep(3000);
  console.log("5");
  setTimeout(wait, 2000);
}, 2000);
// 2 5 2 5 ...
```
2 秒后打印2，sleep 3 秒后打印 5，然后开始下一次调用。时间间隔恰好是 2 秒，而 sleep 函数也可以生效。
