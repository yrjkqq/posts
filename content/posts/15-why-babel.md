---
title: "Babel 究竟用来做什么？"
date: 2021-12-21T10:57:00+08:00
description: "重学前端系列"
draft: false
tags: [重学前端]
---

> 经过这几年的前端工作，我觉得有必要停下来回顾一下这些年用过的前端工具，一方面是学习这些新工具的新的生机，一方面也是对现有知识的梳理，期望找出自身的不足。
> 
> 在网上发现一个资源:[前端路线图](http://codylindley.com/)


## Babel 是什么？

babel 是一个 JavaScript 编译器，它可以转换语法、为不支持的特性添加腻子脚本、源代码转换。


## Babel 解决什么问题？

Babel 可以将浏览器暂不支持的语法通过添加腻子脚本转换为支持的语法，这不仅包括旧浏览器对 es6 的支持也包括最新的浏览器对最新的 es2021 语法的支持，它让我们在写代码时候不需要考虑浏览器支不支持。

1. Babel 通过[插件](https://babeljs.io/docs/en/plugins)使用语法转换来支持最新的 JavaScript 语法，而不用等待浏览器的支持。
2. Babel 能转换 jsx 语法。
3. Babel 支持 TypeScript 语法，能够自动去掉类型注释。
4. Babel 默认不带任何插件。可以使用已有的插件或自己写的插件来管道化你的代码转换。
5. Babel 支持 sourcemap，可以进行 debug.
6. Babel 与最新的 ECMAScript 标准保持一致。为了与标准兼容甚至不惜牺牲性能。
7. Babel 通过 [assumptions(假定)](https://babeljs.io/docs/en/assumptions) 来去掉一些假定代码中不会使用到的代码腻子脚本来减小体积 

## babel 怎么使用？

1. 引入 @babel/polyfill

安装 `yarn add @babel/polyfill` 后在配置文件中增加 `"useBuiltIns": "usage"` 来只引入使用到的 polyfill，`@babel/polyfill` 会自动安装 2.6.12 版本的 `core-js`
```js
// babel.config.js
const presets = [
  [
    "@babel/preset-env",
    {
      targets: {
        edge: "17",
        firefox: "60",
        chrome: "67",
        safari: "11.1",
      },
      useBuiltIns: "usage",
      corejs: "2.6.12",
    },
  ],
];

module.exports = { presets };
```
输出结果如下：
![yarn build output](/posts/img/Snipaste_2021-12-22_11-22-55.png)
从输入结果可以看出，使用 `useBuiltIns: "usage"` 后只引入了使用到的 polyfill

如果没有使用 usage 选项，那就必须安装 `core-js` 和 `regenerator-runtime` 然后在入口文件处引入其他文件之前一次性引入全量的 polyfill，并设置 `"useBuiltIns": "entry"`，如下所示：
```js
import "core-js/stable";
import "regenerator-runtime/runtime";
```
![yarn build output](/posts/img/Snipaste_2021-12-22_11-42-23.png)

`@babel/plugin-transform-runtime` 这个插件用来替换 `@babel/polyfill` 以避免污染全局环境。

2. 不安装 @polyfill

如果仍然需要 core-js，参考[Helpers + polyfilling from core-js](https://babeljs.io/docs/en/v7-migration#helpers--polyfilling-from-core-js) 同时使用 helper 和 polyfill
![yarn build output](/posts/img/Snipaste_2021-12-24_12-32-44.png)



## 其他有用的东西

1. babel 可以[打印出对某个文件生效的配置](https://babeljs.io/docs/en/configuration#print-effective-configs)，也来调试比较方便
![yarn build output](/posts/img/Snipaste_2021-12-22_12-11-42.png)