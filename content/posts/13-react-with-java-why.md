---
title: "React build 时为何会要用到 JDK?"
date: 2020-12-06T12:00:00+08:00
description: "构建时使用 JDK 做什么？"
draft: false
tags: [React]
---

React 官方[贡献指南](https://reactjs.org/docs/how-to-contribute.html#contribution-prerequisites)中提到，运行 React 源代码需要使用到 JDK, 而且在使用 `yarn build` 时，如果没有安装 JDK 则会报错，这是为什么呢？要 Java 做什么呢？本文将一探究竟。

`yarn build` 会运行 `node ./scripts/rollup/build.js` 脚本，运行完会输出: ![yarn build output](/posts/img/Snipaste_2020-12-06_18-08-44.png)

控制台会打印一个表格，这个表格是不是就是用到 Java 相关的库输出的呢？

这个表格是使用 `cli-table` 和 `chalk` 库实现的，与 Java 无关。`printResults` 方法会获取 table 类型的数据。
```js
// scripts\rollup\build.js
async function buildEverything() {
  // ...

  console.log(Stats.printResults());
  if (!forcePrettyOutput) {
    Stats.saveResults();
  }

  if (shouldExtractErrors) {
    console.warn(
      '\nWarning: this build was created with --extract-errors enabled.\n' +
        'this will result in extremely slow builds and should only be\n' +
        'used when the error map needs to be rebuilt.\n'
    );
  }
}

buildEverything();
```

`yarn build` 会同时输出 `react-native` 的包，猜测是构建 rn 库时需要用到 Java. 没有看到相关的代码。

全局搜索 java 关键字，发现有一个包 `google-closure-compiler` 的依赖包中有 java 字样。[google-closure-compiler](https://developers.google.com/closure/compiler/docs/gettingstarted_app) 这个包会使用 java 将 JavaScript 代码进行压缩、优化、查找错误。所以需要安装 JDK.

在 React 源码中，使用 `scripts\rollup\plugins\closure-plugin.js` 这个 rollup 插件进行调用 closure-compiler。
```js
// scripts\rollup\build.js
const closure = require('./plugins/closure-plugin');
function getPlugins(
    // ...
    // Apply dead code elimination and/or minification.
    isProduction &&
      closure(
        Object.assign({}, closureOptions, {
          // Don't let it create global variables in the browser.
          // https://github.com/facebook/react/issues/10909
          assume_function_wrapper: !isUMDBundle,
          renaming: !shouldStayReadable,
        })
      ),
    // ...
}
// scripts\rollup\plugins\closure-plugin.js
'use strict';

const ClosureCompiler = require('google-closure-compiler').compiler;
const {promisify} = require('util');
const fs = require('fs');
const tmp = require('tmp');
const writeFileAsync = promisify(fs.writeFile);

function compile(flags) {
  return new Promise((resolve, reject) => {
    const closureCompiler = new ClosureCompiler(flags);
    closureCompiler.run(function(exitCode, stdOut, stdErr) {
      if (!stdErr) {
        resolve(stdOut);
      } else {
        reject(new Error(stdErr));
      }
    });
  });
}
```