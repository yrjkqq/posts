---
title: "React Hooks 性能优化"
date: 2020-11-13T00:35:00+08:00
description: "探索使用 React Hooks 后常用的优化手段"
draft: false
tags: [Node]
---

React 16.8.0 引入 Hook 之后，新写的项目中基本上全部采用 Hook 的写法。

Hook 与 Class 相比，主要带来以下几方面的优势：
- **复杂组件变得难以理解**  
  Class 组件中需要将各个业务代码写在各个生命周期函数中，可能存在同样的功能分布在各个生命周期函数中，代码逻混乱，尤其是在复杂组件中。而 Hook 将组件中相互关联的部分拆分成更小的函数，而非强制按照生命周期划分。
- **在组件之间复用状态逻辑很难**  
  React 中会使用到 `render props` 或 `高阶组件` 来解决逻辑复用的问题，但这些方式需要改变代码结构，例如使用高阶函数后会产生组件嵌套，层次太多就会产生嵌套地狱。而 Hook 从组件中提取状态逻辑，使得这些逻辑可以单独测试并复用，并且无需修改代码结构。
- **难以理解的 Class**  
  Class 由于 JavaScript 语言的特殊性，在实现方面没有稳定的提案，相比于函数式组件学习成本更高，例如 `this` 的使用或者绑定事件函数。同时，Class 组件会使一些新的优化手段无法在 React 实现，例如组件预编译。而且 Class 不能进行很好的压缩。

Hook 可以在使用函数式组件的情况下使用到更多的 React 特性。但是切换到 Hook 之后，就会就会存在之前的一些优化手段需要使用对应的 Hook 写法。

使用 Class 组件时，常见的优化手段：
- **Component 类组件使用 `shouldComponentUpdate`**    
  可以根据 `shouldComponentUpdate` 的返回值，判断 React 组件的输出是否受当前 state 或 props 更改的影响。在该方法内部进行对变化前后的 props 和 state 进行比较，返回则会跳过更新，即不会吊桶 `render()` 和 `componentDidUpdate()`. 但是返回 false 并不会阻止子组件在 state 变化重新渲染。
- **使用 React.PureComponent 组件**  
  如果手动编写的 shouldComponentUpdate 组件只是对 state 或 props 进行了浅层比较，那么可以直接使用 PureComponent 组件，该组件以浅层比较 state 和 props 的方式实现了 shouldComponentUpdate 函数。React.Component 中的 shouldComponentUpdate() 将跳过所有子组件树的 prop 更新，所以需要确保子组件也是纯组件。
- **使用不可变数据**  
  使用 PureComponent 进行浅层比较时，对象类型例如数组，那么数组添加元素后，对象引用没有变，那么组件不会进行更新。可以使用扩展运算符来返回一个新的对象和数组，即可解决。或者是直接使用 immer 这些库来解决深层嵌套的问题。

再切换到 Hook 之后，常见的优化手段：
- **条件式的跳过 effect**  
- **在依赖列表中省略函数**  
  通常需要在 effect 中声明它所需要的函数，这样比较清楚的看出函数依赖了哪些 state 或 props. 如果无法将函数移动或声明在 effect 内部，有一些其他办法：  
  - 将函数移动到组件之外
  - 在 effect 外调用这个函数，然后让 effect 依赖它的返回值
  - 使用 useCallback 包裹该函数，那么重新渲染前后该函数不会发生改变，那么在 effect 中再依赖这个包裹的函数，即可保证 effect 不会由于函数的变化而发生不必要的调用（函数本身也是一个对象，如果直接依赖未包裹的函数，那么把函数传入子组件并且子组件的 effect 依赖这个函数后，在每次父组件更新时，都会产生一个新的函数，虽然新旧函数完全一致，但浅比较的结果依然不同，那么子组件的 effect 会由于依赖项数组变化依然会被执行）。示例代码：  
    ```js
    function ProductPage({ productId }) {
      // ✅ 用 useCallback 包裹以避免随渲染发生改变
      const fetchProduct = useCallback(() => {
        // ... Does something with productId ...
      }, [productId]); // ✅ useCallback 的所有依赖都被指定了

      return <ProductDetails fetchProduct={fetchProduct} />;
    }

    function ProductDetails({ fetchProduct }) {
      useEffect(() => {
        fetchProduct();
      }, [fetchProduct]); // ✅ useEffect 的所有依赖都被指定了
      // ...
    }
    ```
- **effect 的依赖项频繁变化**  
  使用 [setState 的函数式更新方式](https://zh-hans.reactjs.org/docs/hooks-reference.html#functional-updates) , setState 会接受一个回调函数，该函数将接受之前的 state 便返回更新后的值。那么在 effect 中可以不用直接依赖 state 而内部又可以使用到 state 进行计算, 在 state 变化后不会触发 effect 的重复执行。  
  在更复杂的场景中，可以使用 useReducer 将 state 的更新逻辑移动到 effect 之外。useReducer 的 dispatch 函数永远是稳定的。
- **实现 shouldComponentUpdate**  
  使用 `React.memo` 包括一个组件，memo 会对包裹组件的 props 进行浅比较。等效于 PureComponent，但只比较 props. memo 也接受第二个参数指定一个自定义的比较函数来比较新旧 props，来决定是否跳过更新。
- **useMemo 记忆计算结果**  
  useMemo 允许记住上一次计算结果，在多次渲染之间缓存结果：`const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);`，依赖项数组没有改变，则会跳过二次调用，只返回上一次返回的值。
  使用 useMemo 来优化每一个具体的子结点，并可以跳过子节点的更新，代码如下：
  ```JS
  function Parent({ a, b }) {
    // Only re-rendered if `a` changes:
    const child1 = useMemo(() => <Child1 a={a} />, [a]);
    // Only re-rendered if `b` changes:
    const child2 = useMemo(() => <Child2 b={b} />, [b]);
    return (
      <>
        {child1}
        {child2}
      </>
    )
  }
  ```
- **如何创建昂贵的对象？**  
  [useState 可以接受一个函数作为参数](https://zh-hans.reactjs.org/docs/hooks-reference.html#lazy-initial-state)，这个函数只会在首次渲染时调用这个函数，后续渲染会被忽略。*React 会确保 setState 函数的标识是稳定的，并且不会在组件重新渲染时发生变化，所以可以在 effect 的依赖项中忽略 setState*
- **Hook 会在渲染时因为创建函数而变慢吗？**  
  闭包在极端的场景下可会和类的性能有明显差别。但是 Hook 在某些方面更加高效：  
  - Hook 避免了创建 class 需要的额外开支。
  - 在使用 Hook 时不需要很深的组件树嵌套。相比于使用高阶组件、render props 和 context 更少使用组件嵌套。  
  
  在 React 中使用内联函数对性能的影响，与每次渲染都传递新的回调会破坏子组件的 shouldComponentUpdate 有关。函数本身也是一个对象，在每次渲染时都会重新创建，如果这些函数被传入子组件，而子组件由 PureComponent 或类似实现，那么在比较 props 时，props 中有回调函数，浅比较中前后两次的回调函数会返回 false, 导致子组件重新渲染。而如果子组件中的 effect 依赖了该回调函数，也会被重新执行。解决这个问题有以下三种方法：
  - 使用 useCallback 包裹这个回调函数，那么在重新渲染之间会保持对相同的回调函数的应用，使得子组件的 shouldComponent 或 effect 依赖数组的浅比较得以正常工作
    ```js
    // 除非 `a` 或 `b` 改变，否则不会变
    const memoizedCallback = useCallback(() => {
      doSomething(a, b);
    }, [a, b]);
    ```
  - 使用 useMemo 来控制子节点何时更新，而不直接使用 PureComponent
  - 使用 useReducer，不传递回调函数，而传递 dispatch 函数，因为 dispatch 函数在重新渲染之间是保持稳定的
- **如何避免向下传递回调？**  
  组件树中传递回调函数在写法上更明确，但是管理很麻烦，而且存在上一节中存在的问题。在大型的组件树中，可以通过用 context 用 useReducer 往下传递 dispatch 函数，这样更加方便维护，不用不断转发回调。`dispatch 是处理深度更新的推荐模式。`子组件使用 dispatch 函数向上传递 action 来更新 content.  