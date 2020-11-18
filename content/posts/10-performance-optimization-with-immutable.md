---
title: "React 结合 Immutable 进行性能优化"
date: 2020-11-19T00:08:00+08:00
description: "在 React 中使用 Immutable 进行极致得性能优化"
draft: false
tags: [React]
---

React 中虽然可以使用 `useMemo` `useCallback` `React.memo` 等 api 进行性能优化，但是如果是复杂场景下的复杂数据的更新，使用 [Immutable](https://immutable-js.github.io/immutable-js/) 是更合适的。 

在下面这个[例子](https://codesandbox.io/s/mystifying-pond-zx93e?file=/src/App.tsx)中:
```jsx
import * as React from "react";
import "./styles.css";
import { List } from "immutable";

function ListChild({ names }) {
  const [v, setV] = React.useState("a");
  return (
    <div>
      {Date.now()}-{names.join(",")}
      <input value={v} onChange={(e) => setV(e.target.value)} />
    </div>
  );
}

const MemoListChild = React.memo(ListChild, (prevProps, nextProps) => {
  return prevProps.names.equals(nextProps.names);
});

export default function App() {
  const [value, setValue] = React.useState("");
  const [names, setNames] = React.useState<List<string>>(List([]));

  function handleChange(e) {
    setValue(e.target.value);
    setNames(List(["a"]));
  }

  const memoListChild = React.useMemo(() => <ListChild names={names} />, [
    names
  ]);

  return (
    <div className="App">
      <h1>Hello CodeSandbox</h1>
      <h2>Start editing to see some magic happen!</h2>
      <input value={value} onChange={handleChange} />
      {memoListChild}
      <MemoListChild names={names} />
    </div>
  );
}

```
names 在输入框输入的过程中会不断的更新，每次调用 setNames 时都会生成一个新的对象。在使用 useMemo 时，因为浅比较，内部使用 Object.is 将前后两次的 names 的引用进行比较，自然是返回不相等，导致 memoListChild 变化，在 ListChild 中可以看到时间戳被更新了。

而如果使用不可变数据的专用方法 equals 在 React.memo() 的回调函数中进行比较，虽然重新渲染前后的 names 是不同的对象，但是实际内容是相等的，所以这里返回 true, 即可跳过子组件的更新，达到性能优化目的。

![使用不可变数据前后变化](/posts/img/Snipaste_2020-11-19_00-22-21.png)

而在下面这个例子中，可以看到就算是浅比较也是可以避免更新的，这是由于 immutable 类型变量在使用类似 put set 等方法更新时会返回一个新的对象，但是如果实际内容没有进行更新时，出于性能考虑，仍然会返回原来的对象。所以使用 useMemo 进行浅比较时也返回相等，会跳过更新。示例代码如下：
```jsx
import * as React from "react";
import "./styles.css";
import { Map } from "immutable";

function ListChild({ names }) {
  const [v, setV] = React.useState("a");
  return (
    <div>
      {Date.now()}-{JSON.stringify(names)}
      <input value={v} onChange={(e) => setV(e.target.value)} />
    </div>
  );
}

const MemoListChild = React.memo(ListChild, (prevProps, nextProps) => {
  return prevProps.names.equals(nextProps.names);
});

export default function App() {
  const [value, setValue] = React.useState("");
  const [name, setName] = React.useState<Map<string, string>>(Map({ k: "v" }));

  function handleChange(e) {
    setValue(e.target.value);
    setName(name.set("k", "vv"));
  }

  const memoListChild = React.useMemo(() => <ListChild names={name} />, [name]);

  return (
    <div className="App">
      <h1>Hello CodeSandbox</h1>
      <h2>Start editing to see some magic happen!</h2>
      <input value={value} onChange={handleChange} />
      {memoListChild}
      <MemoListChild names={name} />
    </div>
  );
}
```

[Immutable](https://immutable-js.github.io/immutable-js/) 文档中对这种优化的说明：
> Note: As a performance optimization Immutable.js attempts to return the existing collection when an operation would result in an identical collection, allowing for using === reference equality to determine if something definitely has not changed. This can be extremely useful when used within a memoization function which would prefer to re-run the function if a deeper equality check could potentially be more costly. The === equality check is also used internally by Immutable.is and .equals() as a performance optimization.
> 
> 处于性能优化考虑，Immutable 在更新时如果返回一个同样的集合，那么仍然会返回已存在的对象。这样在使用 === 比较时结果返回相等。