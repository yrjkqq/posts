---
title: "React.isValidElement 如何实现？"
date: 2020-12-06T16:00:00+08:00
description: "isValidElement 实现原理一探究竟"
draft: false
tags: [React]
---

`isValidElement` 函数从 `packages\react\src\ReactElement.js` 导出，其中使用到 $$typeof 判断是否是 ReactElement.
```js
/**
 * Verifies the object is a ReactElement.
 * See https://reactjs.org/docs/react-api.html#isvalidelement
 * @param {?object} object
 * @return {boolean} True if `object` is a ReactElement.
 * @final
 */
export function isValidElement(object) {
  return (
    typeof object === 'object' &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

我们知道，React 中的节点都会使用 React.createElement 创建。`createElement` 会返回一个 ReactElement 
```js
const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };
```
该方法中，将 $$typeof 指定为 REACT_ELEMENT_TYPE 常量值。

回到 isValidElement，即可知道只要是 React 创建的节点，都会有一个参数 $$typeof。
