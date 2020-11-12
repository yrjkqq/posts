---
title: "React Fiber 架构"
date: 2020-11-12T11:12:00+08:00
description: "React 16 引入新的 Fiber 架构"
draft: false
tags: [Node]
---

Fiber 架构主要是使用了新的 reconciler 调谐器，区别于 React 15 的 stack reconciler 调谐器。

Reconciliation 协调主要是是 Diffing 算法。基于两个前提：
- 两个不同类型的元素会产生不同的树
- 开发者可以通过 key prop 来暗示那些子元素在不同的渲染下能保持稳定

启发性算法复杂度由之前的 O(n^3) 将低到 O(n)。

不同类型的元素：根节点不通，直接会卸载原来的树并构建新的树。

对比同一类型的 DOM 元素：保留 DOM 节点，仅比对及更新有改变的属性。

同一类型的组件元素：当一个组件更新时，组件实例保持不变，state 在跨越不同的渲染时保持一致。React 将更新组件实例的 props 以跟最新的元素保持一致，并调用 componentWillUpdate 方法。

下一步调用 render 方法，diff 算法将在之前的结果以及新的结果中进行递归。

key 属性，React 使用 key 属性来匹配原有树上的子元素和最新树上的子元素，匹配到则检测更新，否则就新增结点或删除。没有 key 则会对整个节点树进行比较。

满足假设：
- 类似的组件应该封装成同一类型，有利于 React 进行匹配
- key 保持稳定并唯一。