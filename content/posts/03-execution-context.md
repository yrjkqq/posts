---
title: "JavaScript 的执行上下文和执行栈"
date: 2017-09-08T19:41:00+08:00
description: "复习一些基础数学知识。"
draft: false
tags: [JavaScript]
---

> 参考  
> [Understanding Execution Context and Execution Stack in Javascript](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)  
> 
> [JavaScript深入之词法作用域和动态作用域](https://github.com/mqyqingfeng/Blog/issues/3)  

**执行上下文**是指解释和执行 JavaScript 代码的环境的抽象概念。有以下三种类型：  
1. 全局执行上下文  
未在函数中执行的代码都运行全局执行上下文中，一个程序中只能有一个全局执行上下文。创建 window 对象（浏览器中），将 this 指向 window 对象。

2. 函数执行上下文  
每当函数被调用时，就会为该函数创建一个执行上下文。

3. eval 执行上下文

**执行栈**是一个拥有后进先出结构的栈，用于保存代码执行过程中的执行上下文。

当 JavaScript 引擎开始解析脚本时，会首先创建全局执行上下文，并将其推入当前执行栈。后续每次遇到函数调用时，都会为其创建一个函数执行上下文，并将其推入执行栈栈顶。JavaScript 引擎首先会执行栈顶的函数，执行完该函数出栈，然后继续栈顶函数，直到全局执行上下文。

# 如何创建执行上下文？

分为以下两个阶段：

## 1. 创建阶段  

创建阶段发生了以下两件事：  
- 创建词法环境
- 创建变量环境

概念上可以表示为：
```js
ExecutionContext = {
  LexicalEnvironment = <ref. to LexicalEnvironment in memory>,
  VariableEnvironment = <ref. to VariableEnvironment in  memory>,
}
```

### 词法环境

ES6 中定义：
> 词法环境是一个规范类型，基于 ECMAScript 代码的词法嵌套结构，用于定义标识符和特定的变量和函数的关联。一个词法环境由一个环境记录器和一个可能为 null 的对外部词法环境的引用组成。

简而言之，词法环境是一个标识符-变量的映射结构。

词法环境包括三部分：

#### 环境记录器

词法环境中，使用环境记录器记录变量和函数声明。分为两种类型：  
- 声明环境记录器：存储变量和函数声明。函数代码的词法环境包括一个声明环境记录器。
- 对象环境记录器：全局代码的词法环境包括一个对象式的环境记录器。除了变量和函数声明以外，还存储了全局绑定对象（浏览器中为 windows 对象）。因此对于每一个绑定对象属性（浏览器中包括了浏览器为 window 对象提供的属性和方法），记录中会创建一个新的入口。

对于函数代码，环境变量也包含一个 `arguments` 对象，该对象包括了索引和传递给函数的参数的映射关系和表示传递给该函数的参数的个数的 `length` 属性。  
```js
function foo(a, b) {
  var c = a + b;
}
foo(2, 3);

// argument object
Arguments: {0: 2, 1: 3, length: 2}
```

#### 对外部作用域的引用

持有对外部作用域的引用，意味着 JavaScript 引擎如果在当前词法环境下没有查找到对应的变量，则可以到外部作用域中查找。在函数定义时已经确定(**静态作用域**)，不会在运行时改变，涉及到闭包的使用[JavaScript深入之作用域链](https://github.com/mqyqingfeng/Blog/issues/6)。

#### this 绑定

决定或设置 this 的值。如果是在全局执行上下文中，this 指向 global 对象（浏览器中为 window）。函数执行上下文中，this 的值取决于函数的调用方式。如果是通过对象进行调用，则 this 指向该对象，否则 this 指向全局对象（严格模式下，为 undefined）。

词法环境可以抽象为以下伪代码：  
```js
GlobalExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
    },
    outer: <null>,
    this: <global object>
  }
}
FunctionExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
    }
    outer: <Global or outer function environment reference>,
    this: <depends on how function is called>
  }
}
```

### 变量环境

变量环境也是一个词法环境，不过其环境记录器持有的是在执行上下文中通过变量声明创建的绑定关系。ES6 中词法环境和变量环境的区别在于，前者用于存储函数声明和使用 const 和 let 声明的变量绑定，而后者只用于存储由 var 声明的变量绑定关系。


## 2. 执行阶段  

该阶段完成所有变量的赋值和代码执行。

# 示例

通过以下示例代码理解以上的概念。  
```js
let a = 20;
const b = 30;
var c;
function multiply(e, f) {
 var g = 20;
 return e * f * g;
}
c = multiply(20, 30);
```

以上代码开始执行时，JavaScript 引擎首先创建全局执行上下文去执行全局代码，创建阶段的全局执行上下文就像这样：  
```js
GlobalExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: < uninitialized >,
      b: < uninitialized >,
      multiply: < func >
    },
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

执行阶段，完成变量赋值。全局执行上下文就像这样：  
```js
GlobalExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: 20,
      b: 30,
      multiply: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

以上过程只是创建完成了全局执行上下文，此时该上下文位于执行栈栈底。代码继续执行到调用 multiply 函数，此时会创建一个新的函数执行上下文，用于执行函数代码。此时处于创建阶段的函数执行上下文就行这样：  
```js
FunctionExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: undefined
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

创建函数执行上下文的步骤运行到执行阶段，意味着函数内的变量赋值已经完成，此时的函数执行上下文就像这样：  
```js
FunctionExecutionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: 20
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

此时，函数 multiply 被压入执行栈栈顶进行执行。在函数执行完成后，multiply 函数出栈，返回值存储在变量 c 中，执行栈从栈底取出全局执行上下文继续运行，运行完程序退出。

**注意：** 在创建执行上下文的创建阶段中，let 和 const 定义的变量没有关联到任何值，而 var 声明的变量设置为 undefined.  在创建阶段时 JavaScript 引擎扫描代码用于变量和函数声明，函数声明存储在环境中，而变量初始化为 undefined（使用 var 声明）或者是保持未初始化状态（使用 let 和 const 声明）。所以可以在声明 var 变量前访问到，其值为 undefined, 而访问 let 或 const 声明的变量会报错。即**变量提升**。

**注意：** 在执行阶段，如果 JavaScript 引擎在 let 声明的地方没有找到对应的值，则该变量会被赋值为 undefined.


> 相关代码在 [basic-javascript](https://github.com/yrjkqq/learn-nodejs/blob/master/basic-javascript/environment-record.js)