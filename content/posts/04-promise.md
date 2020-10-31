---
title: "ES2015 标准化的 promise"
date: 2018-09-08T19:41:00+08:00
description: "学习 Promise 的用法和相关"
draft: false
tags: [JavaScript]
---

ES6 规定，Promise 对象是一个构造函数，用来生成 Promise 实例。

Promise 新建后就会立即执行。

Promise 构造函数接受一个函数作为参数，该函数的两个参数分别是 resolve 和 reject。如果调用 resolve 和 reject 时带有参数，那么它们的参数会被传递给回调函数。reject 函数的参数通常是 Error 对象的实例，表示抛出的错误；resolve 函数的参数可以是正常值，也可以是另一个 Promise 实例。

调用 resolve 函数后面的代码也会执行，并且会先于 Promise 实例的 then 回调函数执行。因为立即 resolve 的 Promise 是在本轮事件循环的末尾执行，总是晚于本轮循环的同步任务。  
```js
new Promise((resolve, reject) => {
  resolve(1);
  console.log(2);
}).then(r => {
  console.log(r);
});
// 2
// 1
```

then 方法指定的回调函数在运行中抛出错误，也会被气候的 catch() 方法捕获。Promise 对象的错误具有“冒泡”性质，会一直向后传递，直到被捕获为止。

最佳实践：不要在 then() 方法定义 Reject 状态的回调函数，总是使用 catch 方法。

如果没有使用 catch() 方法指定错误处理的回调函数，Promise 对象抛出的错误不会传递到外层代码，即不会有任何反应。Promise 内部的错误不会影响到 Promise 外部的代码，*Promise 会吃掉错误。* 在服务端可以使用 `process.on("unhandledRejection", callback)` 来捕获到这种错误。  
```js
const someAsyncThing = () => {
  return new Promise((resolve, reject) => {
    resolve(x + 1);
  });
};

someAsyncThing().then(() => console.log("ok"));

setTimeout(() => console.log("after 2 seconds"), 2000);

process.on("unhandledRejection", (err, p) => {
  throw err;
});
```

Promise 内部指定在下一轮事件循环再抛出错误。而到下一轮事件循环时，Promise 的运行已经结束了，所以抛出的错误是在 Promise 函数体外抛出的，会冒泡到最外层，成了未捕获的错误，导致程序异常退出。如果改写成 `reject(new Error('error'))` 则会被 catch 捕获到。
```js
const promise = new Promise((resolve, reject) => {
  // resolve("ok");
  setTimeout(() => {
    throw new Error("error");
    // reject(new Error("error"));
  }, 0);
  // throw new Error("error");
});
promise.then((v) => console.log(v)).catch((e) => console.log("e"));
setTimeout(() => console.log("after 2 seconds"), 2000);

// Uncaught Error
```

### Promise.prototype.finally()

ES2018 引入 finally 方法。不管 Promise 对象最后状态如何，都会执行 finally 方法指定的回调函数。


### Promise.all()

`Promise.all()` 方法用于将多个 Promise 实例包装成一个新的 Promise 实例。只有这多个 Promise 实例状态都变成 fulfilled, Promise.all() 返回的 Promise 实例的状态才会变成 fulfilled.

all 方法可以结合 for...of 循环使用，在循环中创建 Promise 实例，发起异步调用，并将该实例保存在数组中，循环完后数组传递给 all 方法，等待所有 promise 完成得到结果。

如果作为参数的 Promise 实例自己定义了 catch 方法，那么它一旦被 rejected, 并不会触发 Promise.all() 的 catch 方法。因为 rejected 的 Promise 实例被自己的 catch 方法捕获后会返回新的 Promise 实例，该实例执行完 catch 方法后也会变成 resolved 状态。

### Promise.race()

将多个 Promise 实例包装成新的 Promise 实例。只有多个实例中有一个改变状态，那么新的 Promise 实例状态也会改变。  
```js
// 如果 5 秒之内 fetch 方法无法返回结果，变量 p 的状态就会变为 rejected，从而触发 catch 方法指定的回调函数。
const p = Promise.race([
  fetch('/resource-that-may-take-a-while'),
  new Promise(function (resolve, reject) {
    setTimeout(() => reject(new Error('request timeout')), 5000)
  })
]);

p
.then(console.log)
.catch(console.error);
```

### Promise.resolve()

将现有对象转换为 Promise 对象。该方法参数分为四种情况：  

1. 参数是一个 Promise 实例  
不做修改，返回这个实例

2. 参数是一个 thenable 对象  
thenable 对象是指具有 then 方法的对象。Promise.resolve() 方法会将这个对象转为 Promise 对象，然后立即执行 thenable 对象的 then() 方法。转换后的 Promise 实例的 then() 方法可以接收到 thenable 对象执行 then() 方法后传递的参数。
```js
let thenable = {
  then: function(resolve, reject) {
    resolve(42);
  }
};

let p1 = Promise.resolve(thenable);
p1.then(function (value) {
  console.log(value);  // 42
});
```

3. 参数没有 then 方法，或者不是对象  
Promise.resolve() 方法返回一个新的已经 resolved 的 Promise 对象。

4. 不带有任何参数  
直接返回一个 resolved 状态的 Promise 对象。立即 resolve 的 Promise 对象是在本轮事件循环的结束时执行，而不是下一轮事件循环开始时。

### Promise.reject()

返回新的状态为 rejected 的 Promise 实例。

### [Promise.try()](https://es6.ruanyifeng.com/#docs/promise#Promise-try)

提案，stage1 阶段。

