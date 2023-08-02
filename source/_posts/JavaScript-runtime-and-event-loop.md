---
title: JavaScript运行时及事件循环
date: 2023-05-03 16:22:25
tags:
- JavaScript
categories:
- 前端
---

先放个人总结，下文会详细解释：

JavaScript运行时维护了一组代理，每个代理包括：

一组执行上下文集合、执行上下文栈、主线程、worker额外线程、(宏)任务队列、微任务队列。

每个代理由事件循环驱动：

- 在一轮事件循环中，依次执行任务队列中的每一个任务，创建对应的执行上下文，压入执行栈执行。
- 期间如果遇到微任务，只将其添加到微任务队列，然后继续执行任务队列中的任务。
- 当任务队列中的任务执行完毕，执行栈为空时，依次执行微任务队列中的每一个微任务，直到微任务队列为空，允许微任务添加微任务。
- 当所有微任务都被执行完毕后，本轮事件循环结束。
- 下一轮事件循环开启，循环往复。

<!--more-->

## 1.为什么JavaScript是单线程？

作为浏览器脚本语言，JavaScript的主要用途是与用户互动，以及操作DOM。这决定了它只能是单线程，否则会带来很复杂的同步问题。

比如，假定JavaScript同时有两个线程，一个线程在某个DOM节点上添加内容，另一个线程删除了这个节点，这时浏览器应该以哪个线程为准？

因此，为了避免复杂性，从一诞生JavaScript就是单线程。这已经成了这门语言的核心特征，将来也不会改变。

为了利用多核CPU的计算能力，HTML5提出Web Worker标准，允许JavaScript脚本创建多个线程，但是子线程完全受主线程控制，且不得操作`DOM`。

因此，这个新标准并没有改变JavaScript单线程的本质，通过Web Worker创建的新线程中甚至不能获取到`document`和`window`对象(**待测试**)。

## 2.JavaScript运行时

JavaScript运行时维护了一组代理，每个代理包括：

一组执行上下文集合、执行上下文栈、主线程、worker额外线程、(宏)任务队列、微任务队列。

### JavaScript执行上下文(execution context)

当一段JavaScript代码在运行的时候，它实际上是运行在执行上下文中。下面3种类型的代码会创建一个新的执行上下文：

- 全局上下文：是为运行代码主体而创建的执行上下文，即默认的，或者说基础的上下文。任何不在函数内部的代码都在全局上下文中。它会执行两件事：创建一个全局的window对象(浏览器环境下)，并且设置`this`的值等于这个全局对象。一个程序中只会有一个全局上下文
- 函数执行上下文：每个函数会在执行时创建自己的执行上下文，这个上下文就是通常说的“本地上下文”。
- Eval函数执行上下文：使用`eval()`函数也会创建一个新的执行上下文。

每一个上下文在本质上都是一种作用域层级，每个代码段开始执行时都会创建一个新的上下文来运行它。每个上下文创建的时候会被推入执行上下文栈，退出时，从上下文栈中移除并销毁。

### 执行上下文栈

执行栈，也就是其他编程语言中所说的“调用栈”，是一种拥有LIFO(后进先出)的数据结构，被用来存储代码运行时创建的所有执行上下文。

当JavaScript引擎第一次遇到你的脚本时，它会创建一个全局的执行上下文并且压入当前执行栈。每当引擎遇到一个函数调用，它会为该函数创建一个新的执行上下文并压入栈的顶部。

引擎会执行处于栈顶的执行上下文的函数。当该函数执行结束时，执行上下文从栈中弹出，控制流程到达当前栈中的下一个上下文。

```javascript
let a = "Hello World!"

function first(){
  console.log('Inside first function')
  second()
  console.log('Again inside first function')
}

function second(){
  console.log('Inside second function')
}

first()
console.log('Inside Global Execution Context')
```

![img](https://raw.githubusercontent.com/w1043545633/imgForPicGo/main/202304061506066.png)

### JavaScript运行时

在执行JavaScript代码的时候，JavaScript运行时实际上维护了一组用于执行JavaScript的代理。每个代理由以下几个部分构成：

- 一组执行上下文的集合
- 执行上下文栈
- 主线程
- 一组可能创建用于执行worker的额外的线程集合
- 一个任务队列
- 一个微任务队列

除了主线程(某些浏览器在多个代理之间共享主线程)之外，其他组成部分对该代理都是唯一的。

介绍了这么多，那么JavaScript运行时究竟是如何工作的？这就不得不提到事件循环了。

## 3.事件循环(Event Loop)

JavaScript运行时维护的每个代理都是由事件循环驱动的。事件循环负责收集事件(包括用户事件以及其他非用户事件)、对任务进行排队以便在合适的时候执行回调。

事件循环首先执行所有处于等待中的JavaScript任务(宏任务)，然后执行微任务(microtask)，最后在开始下一次循环之前执行一些必要的渲染和绘制操作。

网页或者app的代码和浏览器本身的用户界面程度运行在相同的线程中，共享相同的事件循环，该线程即为主线程。主线程除了运行网页本身的代码之外，还负责收集和派发用户和其他事件，以及渲染和绘制网页内容等等。

### 宏任务vs微任务

注意：宏任务在MDN及一些文档中，被称为JavaScript任务

宏任务包括：

- script(整体代码)，由标准机制来执行的任何JavaScript
- setTimeout、setInterval、setImmediate
- I/O
- UI rendering

微任务包括：

- process.nextTick
- Promise
- Object.observe(已废弃)
- MutationObserver(html5新特性)

### 事件循环步骤

现在我们来更新前文提到的事件循环步骤，主线程也是从宏任务开始的：

- 执行一个宏任务
- 执行过程中如果遇到微任务，加入微任务队列；遇到宏任务，加入(下一个)宏任务队列
- 当前队列的宏任务执行完毕后，依次执行微任务队列。微任务执行完毕后，一轮事件循环结束
- 开始下一个宏任务

每一次新的事件循环开始迭代的时候，JavaScript运行时都会执行宏任务队列中的每个任务。在每次迭代开始之后加入到队列中的任务需要在下一次迭代开始之后才会被执行。

每次当一个宏任务退出且执行上下文为空时，微任务队列中的每一个微任务会依次被执行。不同的是，它会等到微任务队列为空时才会停止执行————即使中途有微任务加入。换句话说，微任务可以添加新的微任务到队列中，并且在下一个宏任务开始执行之前，且当前事件循环结束之前执行完所有的微任务。

### 来一道面试题加深理解

说出下面代码的运行结果，并说明原因：

```javascript
async function async1(){
    console.log('async1 start')
    await async2()
    console.log('async1 end')
}

async function async2(){
    console.log('async2')
}

console.log('script start')

setTimeout(function(){
    console.log('setTimeOut')
}, 0)

async1()

new Promise(function(resolve){
    console.log('promise1') 
    resolve()
}).then(function(){
    console.log('promise2') 
})

console.log('script end')
```

结果如下：

```
//script start
//async1 start
//async2
//promise1
//script end
//async1 end
//promise2
//setTimeOut
```

这段代码完美的体现了宏任务与微任务的执行区别，让我们来详细分析一下执行流程：

1. 第一轮事件循环开始
2. 打印script start
3. 遇到setTimeout，放入(下一轮)宏任务中，等待执行
4. 遇到async1，同步执行，打印async1 start
5. 遇到async2，同步执行，打印async2
6. 将await async2函数后面的回调函数(async1 end)放入微任务队列
7. 遇到new Promise，同步执行，打印promise1，将其回调函数(promise2)放入微任务队列
8. 打印script end，宏任务执行完毕，开始执行微任务
9. 打印async1 end
10. 打印promise2，本轮事件循环结束
11. 进入下一轮事件循环，执行setTimeout，打印setTimeOut

容易产生困惑的地方就在async和await，async函数本质上还是基于Promise的一些封装，将async1函数用Promise进行如下改写，更易于理解：

```javascript
async function async1(){
  console.log('async1 start')
  new Promise(function(resolve){
    console.log('async2')
    resolve()
  }).then(function(){
    console.log('async1 end')
  })
}
```

## 4.NodeJS中的process.nextTick

在Node.js中，每当事件循环进行一次完整的循环时，我们将其称为一次tick。当一个函数传给`process.nextTick()`时，指示引擎在当前操作结束，下一个事件循环tick开始之前调用该函数。

`process.nextTick`回调函数会被加入`process.nextTick`队列，`Promise.then()`回调函数会被加入`promise.microtask`队列(微任务队列)，而`setTimeout`和`setImmediate`回调函数会被加入macrotask队列(宏任务队列)。

三种队列的优先级排序为：

process.nextTick ＞ promise.microtask ＞ macrotask(因为在下一轮事件循环中)

## 参考资料

1. [从一道面试题来理解JS事件循环](https://xieyufei.com/2019/12/30/Quiz-Eventloop.html)
2. [深入：微任务与JavaScript运行时环境 - MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)
3. [(ES5版)深入理解JavaScript执行上下文和执行栈](https://cloud.tencent.com/developer/article/1604839)
4. [并发模型与事件循环 - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Event_loop)
5. [Understanding setImmediate() - Node.js](https://nodejs.dev/en/learn/understanding-setimmediate/)
