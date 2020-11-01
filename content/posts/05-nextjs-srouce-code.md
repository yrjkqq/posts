---
title: "Next.js 源码阅读"
date: 2020-05-12T09:45:00+08:00
description: "Next.js 的源码值得一读"
draft: false
tags: [JavaScript, Node]
---

# 前言

Next.js 官方文档不是很详细，一些细节部分需要深入到源码中，才能知其所以然。按照[官方贡献指南](https://github.com/vercel/next.js/blob/canary/contributing.md) Fork 项目到自己的仓库，然后 clone 下来。以最新的 canary 分支[版本 10.0.1-canary.4](https://github.com/vercel/next.js/commit/4dbf0d47b0c84b9f39adfaf826d5ee5888c514f7) 为基础。

# 项目结构

深入到源码之前，先摸清楚整个项目的结构。

源码中使用 `lerna` 在 packages 目录下放置了多个包，每个包有自己的 package.json 文件。使用类似 `learn run command` 的命令时，会同时运行 packages 目录所有包中可以运行该 command 的命令。

目录结构：  
- `.vscode` 目录下放置了 vscode 相关的配置文件。
- `bench` 目录放置了运行 benchmarks 的代码和文档。
- `docs` 目录下是官方文档。
- `errors` 目录下使用了 Next.js 过程中可能会遇到的错误，记录了错误发生的原因和解决方法。
- `examples` 目录下放置了 Next.js 的用法，或是与各种第三方库或框架结合使用的样例代码
- `packages` 源代码目录，后面主要是阅读这里面的代码
- `test` 测试代码和测试用例

其他比较重要的文件：  
- `.eslintrc.json` ESLint 的配置文件
- `.prettierrc.json` Prettier 的配置文件
- `jest.config.js` 测试框架 jest 的配置文件
- `lerna.json` lerna 的配置文件
- `package.json` 包描述文件，记录了所有可以运行的脚本


# 阅读前准备

按照官方贡献指南，安装完依赖后，运行 `node --inspect packages/next/dist/bin/next dev test/integration/basic` 没有报错，然后打开 `http://localhost:3000/about` 查看页面正常，就安装运行完成。

## 如何启动 Debug 模式？

仓库下 .vscode 目录下有 vscode debugger 的配置文件，可以直接运行 debugger 程序，方便调试。运行 `Launch app development` debugger 程序，即可使用 vscode 的 debug 模式。

打开 `packages\next\bin\next.ts` 文件设置断点，F5 启动调试，进入断点则 debug 模式启动成功。

如果需要修改代码，需要单独运行 `yarn dev` 监听文件的变化并自动编译。修改代码保存后，重新编译完成后需要重新 F5 启动调式程序，才可看到修改后的内容。 

## 如何阅读源代码？

原则：**带着问题进行阅读阅读**，将大问题不断分解成小问题，期间遇到不懂或不清楚的内容，及时记录下来问题和实时阅读进度，待问题解决后再回到之前阅读中断的地方继续。

## 问题1：  
从运行 `node --inspect packages/next/dist/bin/next dev test/integration/basic` 到打开 `http://localhost:3000/about` 查看 about 页面，页面显示 `About Page` 中间发生了什么？可以规划为多少步？

## 问题2：  
Next.js 如何读取项目配置下的 next.config.js 文件的？其中 webpack 的配置是怎么生效的？