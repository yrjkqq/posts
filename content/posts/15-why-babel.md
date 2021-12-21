---
title: "Babel 究竟用来做什么？"
date: 2021-12-21T10:57:00+08:00
description: "重学前端系列"
draft: false
tags: [重学前端]
---

> 经过这几年的前端工作，我觉得有必要停下来回顾一下这些年用过的前端工具，一方面是学习这些新工具的新的生机，一方面也是对现有知识的梳理，期望找出自身的不足。
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
