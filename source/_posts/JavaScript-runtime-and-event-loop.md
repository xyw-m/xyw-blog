---
title: JavaScript运行时及事件循环
date: 2023-05-08 16:22:25
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

## 2.JavaScript运行时环境和执行上下文

### 运行时环境

JavaScript是一门解释执行语言。这意味着源代码在执行前，无需编译为二进制文件。那么问题来了，计算机如何读懂并运行这些纯文本的源代码？

这就是JavaScript引擎(JavaScript engine)的作用，它负责将源代码翻译为机器码，并通过CPU来执行翻译后的机器码。JavaScript引擎可以被视作一个运行你的程序(代码)的容器。

主流的JavaScript引擎包括Chrome V8(谷歌浏览器)、SpiderMonkey(Firefox 浏览器)、Nitro(Safari 浏览器，或者是Webkit)、Chakra(IE、Edge)等。

在Web开发中，开发者并不会直接使用JavaScript引擎，JavaScript引擎是运行在一个环境中的，这个环境提供了代码在执行时能够使用的一些附加特性，比如支持引擎与周围的环境进行交互的工具类库或者API等。这个环境就是JavaScript运行时环境(JavaScript runtime)。

在实际执行JavaScript代码的时候，JavaScript运行时环境维护了一组用于执行JavaScript代码的代理。每个代理由以下部分组成：

- 一组执行上下文集合
- 执行上下文栈
- 主线程
- worker额外线程
- (宏)任务队列
- 微任务队列



### 执行上下文(execution context)

> 本节《执行上下文》及下一节《执行上下文栈》是对运行时环境的深入介绍，MDN对此的建议是：对于大多数开发人员来说，这些细节并不重要。
>
> 如果你不关心这些内容，可以跳过这部分或者在你需要的时候再倒回来查看。

当一段JavaScript代码在运行的时候，它实际上是运行在执行上下文中。下面3种类型的代码会创建一个新的执行上下文：

- 全局上下文：是为运行代码主体而创建的执行上下文，即默认的，或者说基础的上下文。任何不在函数内部的代码都在全局上下文中。它会执行两件事：创建一个全局的window对象(浏览器环境下)，并且设置`this`的值等于这个全局对象。一个程序中只会有一个全局上下文
- 函数执行上下文：每个函数会在执行时创建自己的执行上下文，这个上下文就是通常说的“本地上下文”。
- Eval函数执行上下文：使用`eval()`函数也会创建一个新的执行上下文。

每一个上下文在本质上都是一种作用域层级，每个代码段开始执行时都会创建一个新的上下文来运行它。每个上下文创建的时候会被推入执行上下文栈，退出时，从上下文栈中移除并销毁。

### 执行上下文栈

执行栈，也就是其他编程语言中所说的“调用栈”，是一种拥有LIFO(后进先出)的数据结构，被用来存储代码运行时创建的所有执行上下文。

当JavaScript引擎第一次遇到你的脚本时，它会创建一个全局的执行上下文并且压入当前执行栈。每当引擎遇到一个函数调用，它会为该函数创建一个新的执行上下文并压入栈的顶部。

引擎会执行处于栈顶的执行上下文的函数。当该函数执行结束时，执行上下文从栈中弹出，控制流程到达当前栈中的下一个上下文。

看看下面这段代码：

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

图中所示为上述代码的执行上下文栈。当这段代码在浏览器加载时：

1. JavaScript引擎创建了一个全局执行上下文并把它压入当前执行栈
2. 当遇到`first()`函数调用时，JavaScript引擎为该函数创建一个新的函数执行上下文，并把它压入当前执行栈的顶部
3. 当从`first()`函数内部调用`second()`函数时，JavaScript引擎为`second()`函数创建一个新的函数执行上下文，并把它压入当前执行栈的顶部
4. 当`second()`函数执行完毕，它的执行上下文会从当前栈弹出并销毁，然后控制流程到达下一个执行上下文，即`first()`函数的执行上下文
5. 当`first()`函数执行完毕，它的执行上下文从栈中弹出并销毁，控制流程到达全局执行上下文。一旦所有代码执行完毕，JavaScript引擎从当前栈中弹出并销毁全局执行上下文

## 3.事件循环(Event Loop)

上文已经说过，JavaScript代码是在单一线程中执行的。但是，这并不代表整个JavaScript运行时环境是在单一线程中工作的。JavaScript运行时是存在**线程池**(thread pool)的。幸运的是，你不需要手动进行线程调度，因为JavaScript运行时环境已经帮你做到了这点。

那么JavaScript运行时是如何工作的？如何管理代码执行的顺序？这就是接下来要介绍的重点了 ———— JavaScript中大名鼎鼎的事件循环机制。

JavaScript运行时所维护的代理是由事件循环驱动的——事件循环负责收集事件（包括用户事件以及其他非用户事件等）、对任务进行排队以便在合适的时候执行回调。

在一次事件循环中，首先执行所有处于等待中的JavaScript任务(宏任务)，然后执行微任务(microtask)，最后在开始下一次循环之前执行一些必要的渲染和绘制操作。

### (宏)任务(Tasks)

> 注意：宏任务在MDN及一些文档中，被称为JavaScript任务。这里的”宏“只是为了与微任务队列做出区分。

一个任务就是指计划由标准机制来执行的任何JavaScript，如程序的初始化、触发回调的事件等。除了使用事件，你还可以使用`setTimeout()`，或者`setInterval()`来添加任务。

宏任务包括：

- script(整体代码)，由标准机制来执行的任何JavaScript
- setTimeout、setInterval、setImmediate
- I/O
- UI rendering

在以下时机，任务会被添加到任务队列：

- 一段新程序或子程序被直接执行时(比如从一个控制台，或者在一个`<script>`元素中运行代码)
- 触发了一个事件，将其回调函数添加到任务队列
- 执行到一个`setTimeout()`或`setInterval()`创建的timeout或interval，相应的回调函数被添加到任务队列

事件循环按这些任务排队的顺序，一个接一个地处理它们。在当前迭代轮次中，只有那些当事件循环过程开始时，**已经处于任务队列中**的任务会被执行，其余任务将等到下一次迭代开始之后执行。

### 微任务(Microtasks)

一个微任务就是一个简短的函数，当创建该微任务的函数执行之后，当JavaScript调用栈为空时，而控制权尚未返还给(被用户代理用来驱动脚本执行环境)

微任务包括：

- process.nextTick
- Promise
- Object.observe(已废弃)
- MutationObserver(html5新特性)
- 通过queueMicrotask()添加的微任务

当本轮事件循环的任务队列为空时（执行上下文栈为空），开始执行微任务队列中的任务。微任务会按照它们被添加的顺序来依次执行。

但与任务队列不同的是：总是会等到微任务队列为空时才会停止执行——即使中途有微任务加入。换句话说，允许微任务添加新的微任务到队列中，并且这些新增的微任务也会在本轮事件循环中执行。

### 事件循环步骤

现在我们来更新前文提到的事件循环步骤，主线程也是从宏任务开始的：

- 执行一个宏任务
- 执行过程中如果遇到微任务，加入微任务队列；遇到宏任务，加入(下一个)宏任务队列
- 当前队列的宏任务执行完毕后，依次执行微任务队列。微任务执行完毕后，一轮事件循环结束
- 开始下一个宏任务

每一次新的事件循环开始迭代的时候，JavaScript运行时都会执行宏任务队列中的每个任务。在每次迭代开始之后加入到队列中的任务需要在下一次迭代开始之后才会被执行。

每次当一个宏任务退出且执行上下文为空时，微任务队列中的每一个微任务会依次被执行。不同的是，它会等到微任务队列为空时才会停止执行————即使中途有微任务加入。换句话说，微任务可以添加新的微任务到队列中，并且在下一个宏任务开始执行之前，且当前事件循环结束之前执行完所有的微任务。

## 4. 题外话：何时应该使用微任务

应该使用微任务的场景一般为：捕捉或检查结果、执行清理等。场景需求的时机晚于一段JavaScript执行上下文主体的退出，但早于任何事件处理函数、timeouts或intervals及其他回调被执行。

简单来说，就是确保任务顺序的一致性，因为微任务总是会按照它们被添加进(微任务)队列的顺序来执行。

### 场景1：保证条件性使用promises时的顺序

> 本例来源：[MDN - 在 JavaScript 中通过 queueMicrotask() 使用微任务](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide)

当promise被用在一个`if...else`语句，比如在没有本地缓存时向后端请求数据，有缓存时直接读取本地缓存的业务场景中：

```javascript
customElement.prototype.getData = url => {
  if (this.cache[url]) {
    this.data = this.cache[url];
    this.dispatchEvent(new Event("load"));
  } else {
    fetch(url).then(result => result.arrayBuffer()).then(data => {
      this.cache[url] = data;
      this.data = data;
      this.dispatchEvent(new Event("load"));
    )};
  }
};

element.addEventListener("load", () => console.log("Loaded data"));
console.log("Fetching data...");
element.getData();
console.log("Data fetched");
```

连续执行两次这段代码会形成下表中的结果：

数据未缓存的结果（左）vs. 缓存中有数据的结果

| 数据未缓存    | 缓存中有数据  |
| ------------- | ------------- |
| Fetching data | Fetching data |
| Data fetched  | Loaded data   |
| Loaded data   | Data fetched  |

解决办法是在`if`子句中使用一个微任务来确保操作顺序的一致性：

```javascript
customElement.prototype.getData = url => {
  if (this.cache[url]) {
    // 使用queueMicrotask添加微任务
    queueMicrotask(() => {
      this.data = this.cache[url];
      this.dispatchEvent(new Event("load"));
    });
  } else {
    fetch(url).then(result => result.arrayBuffer()).then(data => {
      this.cache[url] = data;
      this.data = data;
      this.dispatchEvent(new Event("load"));
    )};
  }
};
```

### 场景2：批量操作

> 本例来源：[MDN - 在 JavaScript 中通过 queueMicrotask() 使用微任务](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide)

也可以使用微任务从不同来源将多个请求收集到单一的批处理中，从而避免对处理同类工作的多次调用可能造成的性能开销。

下面的代码创建了一个函数，将多个消息放入一个数组中批处理，并通过一个微任务在上下文退出时将这些消息作为单一的对象发送出去：

```javascript
const messageQueue = [];

let sendMessage = (message) => {
  messageQueue.push(message);

  if (messageQueue.length === 1) {
    queueMicrotask(() => {
      const json = JSON.stringify(messageQueue);
      messageQueue.length = 0;
      fetch("url-of-receiver", json);
    });
  }
};
```

当sendMessage被调用时，指定的消息首先被推入消息队列数组messageQueue。接着事情就变得有趣了：

如果我们刚加入数组的消息是第一条，那么就入列一个将会发送一个批处理的微任务；接下来，等JavaScript执行路径到达顶层，执行上下文退出时（本轮事件循环结束前），那个微任务将会执行。

在这个间歇期内对sendMessage的任务调用都会将其各自的消息推入消息队列，但囿于入列微任务逻辑之前的数组长度检查，不会有新的微任务入列。

当微任务运行之时，等待它处理的可能是一个有若干条消息的数组，处理完成后，清空消息队列messageQueue。

这使得同一次事件循环迭代期间发生的每次`sendMessage()`调用都会将其消息添加到同一个fetch操作中，而不会让如此timeouts等其他可能的定时任务推迟传递。

注：`Vue.$nextTick()`的机制也类似，利用微任务机制在本轮事件循环迭代中统一运行所有传递给`$nextTick()`的回调函数。

另外一个有助于（作者个人）理解的小例子：

```javascript
const messageArr = []

function sendMessage(msg){
    messageArr.push(msg)
    
    if(messageArr.length === 1){
        queueMicrotask(() => {
            for(let str of messageArr){
                console.log(`this tikc of ${str}`)
            }
            console.log('this tick end')
            messageArr.length = 0
        })
    }
}

function fn(){
    console.log('fn run')
    const max = count
    for(let i = count; i < max + 10; i++){
        count++
        sendMessage(i)
    }
}

fn()

setTimeout(function() {
    console.log('setTimeout begin')
    fn()
    console.log('setTimeout end')
}, 10);
```

运行结果：

```
fn run
this tikc of 0
this tikc of 1
this tikc of 2
this tikc of 3
this tikc of 4
this tikc of 5
this tikc of 6
this tikc of 7
this tikc of 8
this tikc of 9
this tick end
setTimeout begin
fn run
setTimeout end
this tikc of 10
this tikc of 11
this tikc of 12
this tikc of 13
this tikc of 14
this tikc of 15
this tikc of 16
this tikc of 17
this tikc of 18
this tikc of 19
this tick end
```

## 5. 最后来一道面试题加深理解

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

## 6. 题外话2：NodeJS中的process.nextTick

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
6. [The JavaScript runtime environment](http://dolszewski.com/javascript/javascript-runtime-environment/)
7. [【译文】扒一扒JavaScript运行时环境](https://zhuanlan.zhihu.com/p/350016161)
8. [在 JavaScript 中通过 queueMicrotask() 使用微任务 - MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide)

