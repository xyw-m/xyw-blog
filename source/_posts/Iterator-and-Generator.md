---
title: 迭代器与生成器
date: 2023-06-10 16:22:25
author: xyw
tags:
- JavaScript
categories:
- 前端
---

JavaScript表示“集合”的数据结构有：`Array`、`Object`、`Map`和`Set`(后两者是ES6新增的)，因此需要一种统一的接口机制，来处理所有不同的数据结构。

<!--more-->

## 1. Iterator与for...of

### Iterator接口

遍历器(Iterator)因此应运而生，它作为接口，为各种不同的数据结构提供统一的访问机制(即`for...of`循环，详见下文)。Iterator的遍历过程(或者说是规范)是这样的：

1. 创建一个指针对象，指向当前数据结构的起始位置。其实遍历器对象本质上，就是一个指针对象。
2. 第一次调用指针对象的`next`方法，将指针指向数据结构的第**一**个成员
3. 第二次调用指针对象的`next`方法，指针指向数据结构的第**二**个成员
4. 以此类推，不断调用`next`方法，直到它指向数据结构的结束位置

每一次调用`next`方法，都会返回数据结构的当前成员的信息。具体来说，就是返回一个包含`value`和`done`两个属性的对象。其中，`value`属性是当前成员的值，`done`属性是一个布尔值，表示遍历是否结束。

ES6规定，默认的`Iterator`接口部署在数据结构的`Symbol.iterator`属性上，一个数据结构只要具有`Symbol.iterator`属性，就认为是"可遍历的"(iterable)。`Symbol.iterator`属性本身是一个函数，即当前数据结构默认的遍历器生成函数。执行这个函数，就会返回一个遍历器(简单来说，就是一个具有`next()`方法的对象)。

ES6中原生具备`Iterator`接口的数据结构有：`Array`、`Map`、`Set`、`String`、`TypedArray`(描述了底层二进制数据缓冲区的类数组对象)、函数的`arguments`对象和`NodeList`对象。

正常情况下，即使已经有了原生`Iterator`接口，也需要(一遍遍地)手动调用`next()`方法来遍历数据结构，这实在是太麻烦了。所以ES6新增了一种遍历命令`for...of`，当使用`for...of`循环遍历某种数据结构时，该循环会自动去寻找`Iterator`接口，也就是说，不用再自己手动调用`next()`方法了！

### 其他用法

那么，除了为`for...of`循环服务之外，`Iterator`接口还有别的用处吗？或者说，还有别的场合会调用`Iterator`接口吗？其实是有的：

1. 解构赋值：对数组和`Set`解构进行解构赋值时，会默认调用`Iterator`接口
2. 扩展运算符(...)
3. `yield*`
4. 任何接受数组作为参数的场合，比如：`Array.from()`、`Map()`、`Promise.all()`等等

这样看来，原生具有`Iterator`接口的数据结构，可以很方便的进行遍历和进一步的操作。但是，如果想遍历原生不具备`Iterator`接口的数据结构呢？比如想用`for...of`遍历对象，那就只能手动在`Symbol.iterator`属性上部署遍历器生成方法(原型链上的对象具有该方法也可)，如下所示：

```javascript
class RangeIterator {
  constructor(start, stop){
    this.value = start
    this.end = stop
  }
  
  [Symbol.iterator](){
    return this
  }
  
  next(){
    const value = this.value
    if(value < this.stop){
      this.value++
      return { value: value, done: false }
    }
    return { value: undefined, done: true }
  }
}

for(let value of new RangeIterator(0, 3)){
  console.log(value)	// 0, 1, 2
}
```

这里还有一个取巧的法子，为类似数组的对象(存在数值键名和`length`属性)部署`Iterator`接口，可以直接使用数组的`Iterator`接口。

### 生成器的妙用

除了上述的特例外，手动部署`Iterator`接口过于繁琐，因此ES6同时新增了生成器(Generator)，用于自定义遍历器和实现协程。

上边那个例子使用生成器可以改写成这样，极大地简化了手动部署`Iterator`接口的操作！

```javascript
class RangeIterator {
  constructor(start, stop){
    this.value = start
    this.end = stop
  }
  
  *[Symbol.iterator](){
    for(let i = this.value; i < this.stop; i++){
      yield i
    }
  }
}

for(let value of new RangeIterator(0, 3)){
  console.log(value)	// 0, 1, 2
}
```

下面我们会详细介绍一下生成器Generator。

## 2. Generator函数

从语法上来理解，Generator函数是一个状态机，封装了多个内部状态。此外，它还是一个遍历器生成函数，执行它会返回一个遍历器对象。(还记得吗，可迭代数据结构的`Symbol.iterator`属性也是一个遍历器生成函数)

从形式上来说，Generator函数是一个普通函数，但有两点不同：

1. `function`关键字与函数名之间有一个星号`*`
2. 使用`yield`**表达式**，定义不同的内部状态

运行Generator函数会返回一个指向内部状态的指针对象，也就是前面介绍过的遍历器对象。遍历器对象的`next`方法的运行逻辑如下：

1. 遇到`yield`表达式，暂停执行后面的操作，并将紧跟在`yield`后面那个表达式的值，作为返回的对象的`value`属性值
2. 下一次调用`next`方法时，再继续往下执行，直到遇到下一个`yield`表达式
3. 如果没有再遇到新的`yield`表达式，就一直运行到函数结束，直到`return`语句为止，并将`return`语句后面的表达式，作为返回的对象的`value`属性值
4. 如果该函数没有`return`语句，则返回的对象的`value`属性值为`undefined`

注意：`yield`表达式后面的表达式，只有当调用`next`方法，内部指针指向该语句时才会执行。

### 自定义`Iterator`接口

Generator函数的典型应用场景就是可以在任意对象上轻松部署`Iterator`接口，然后对象就可以被`for...of`命令迭代了！

除此之外，因为脚本遇到`yield`就暂行执行，这意味着Generator函数可以作为一种异步解决方案，把异步操作写在`yield`表达式里面，等到后续调用`next()`方法时再往后执行，这实际上等同于不需要写回调函数了，`yield`表达式后面的操作就等同于回调函数。

更进一步讲，如果有一个多步操作非常耗时，采用回调函数，可能会导致著名的“回调函数地狱”。即使用Promise改写，代码的语义也很不清楚。而Generator函数可以进一步改善代码的运行流程，如下所示：

(注意，这种做法只适合同步操作，因为这里的代码一得到返回值，就继续往下执行，没有判断异步操作何时完成。异步操作流程详见下文)

```javascript
/* 回调函数地狱 */
step1(function (value1) {
  step2(value1, function(value2) {
    step3(value2, function(value3) {
      step4(value3, function(value4) {
        // Do something with value4
      });
    });
  });
});

/* Promise改写 */
Promise.resolve(step1)
  .then(step2)
  .then(step3)
  .then(step4)
  .then(function (value4) {
    // Do something with value4
  }, function (error) {
    // Handle any error from step1 through step4
  })
  .done();

/* Generator优化 */
function* longRunningTask(value1) {
  try {
    var value2 = yield step1(value1);
    var value3 = yield step2(value2);
    var value4 = yield step3(value3);
    var value5 = yield step4(value4);
    // Do something with value4
  } catch (e) {
    // Handle any error from step1 through step4
  }
}

/* 手动编写函数，按次序自动执行所有步骤 */
function scheduler(task){
  const taskObj = task.next(task.value)
  if(!taskObj.done){
    task.value = taskObj.value
    scheduler(task)
  }
}
scheduler(longRunningTask(initialValue))
```

## 3. Generator函数的异步应用

Generator函数遇到`yield`就暂停执行，调用`next()`就恢复执行的特性，使得它可以作为 JavaScript 中的异步解决方案。

ES6诞生以前，JavaScript异步编程的方法，大概有四种：回调函数、事件监听、发布/订阅和Promise对象，ES6新增的Generator函数带来了新的异步方案。

在其他传统的编程语言中，早有许多异步编程的解决方案。其中有一种叫做“协程”(coroutine)，意为多个线程相互协作，完成异步任务。它的运行流程大致如下：

1. 协程A开始执行
2. 协程A执行到一半，进入暂停，执行权转移到协程B
3. (一段时间后)协程B交还执行权
4. 协程A恢复执行

JavaScript是单线程语言，所以此处的协程A可以等同于异步任务A。我们不难看出，Generator函数采用了协程的思想，整个Generator函数可以看做异步任务的容器，异步操作需要暂停的地方，都用`yield`语句注明。

除了暂停执行(`yield`)和恢复执行(`next()`)，Generator函数还支持函数体内外的交换和错误处理机制 ：

- `next`返回值的`value`属性，是Generator函数向外**输出**数据，`next`方法接受的参数，是向Generator函数体内**输入**数据。
- `throw`方法可以在函数体外抛出错误，然后在函数体内捕获

### 实例：执行异步任务

下面来看如何使用Generator函数，执行一个真实的异步任务：

```javascript
let fetch = require('node-fetch')

function* gen(){
  let url = 'https://api.github.com/users/github'
  let result = yield fetch(url)
  console.log(result.bio)
}

let g = gen()
let result = g.next()
result.value.then(function(data){
  return data.json()
}).then(function(){
  g.next(data)
})
```

上面代码中，首先执行Generator函数，获取遍历器对象，然后使用`next`方法，执行异步任务的第一阶段，向远程服务器发送请求。由于`Fetch`模块返回的是一个`Promise`对象，因此要用`then`方法拿到请求的返回值，转换为`json`后再作为参数传入`next`方法中，最终被打印到控制台上。

可以看到，虽然Generator函数将异步操作表示得很简洁，但是流程管理却很不方便，我们很难控制何时执行第一阶段，何时执行第二阶段。我们希望有一种方法，当判断异步操作执行完成时，可以自动执行`next`方法，将执行权交还给Generator函数。

自动执行的关键是：有一种机制，自动控制Generator函数的流程，接收和交还程序的执行权。两种方法可以做到这点：

1. 回调函数：将异步操作包装成Thunk函数，在回调函数里面交回执行权
2. Promise对象：将异步操作包装成Promise对象，用then方法交回执行权

### 自动执行Generator函数的模块

#### thunkify

在编译器中，Thunk函数用于实现“传名调用”，即将参数放到一个临时函数之中，再将这个临时函数传入函数体，这个临时函数就叫做Thunk函数。

而JavaScript语言是传值调用，所以它的Thunk函数替换的不是表达式，而是多参数函数，将其替换成一个只接受回调函数作为参数的单参数函数。

任何函数，只要参数有回调函数，就能写成Thunk函数的形式，下面是一个简单的Thunk函数转换器：

```javascript
/* ES5版本 */
var Thunk = function(fn){
  return function(){
    let args = Array.prototype.slice.call(arguments)
    return function(callback){
      args.push(callback)
      return fn.apply(this, args)
    }
  }
}

/* ES6版本 */
const Thunk = function(fn){
  return function(...args){
    return function(callback){
      return fn.call(this, ...args, callback)
    }
  }
}

/* 生成fs.readFile的Thunk函数 */
const readFileThunk = Thunk(fs.readFile)
readFileThunk(fileA)(callback)
```

[Thunkify模块](https://www.npmjs.com/package/thunkify)可以将常规函数转换为thunk函数。

[thunk函数的原理](https://www.bookstack.cn/read/es6-3rd/spilt.4.docs-generator-async.md#cw2ht4)此处不再详细介绍，只需要知道利用thunk函数可以自动执行Generator函数。

下面是一个基于Thunk函数的Generator执行器：

```javascript
function run(fn){
  let gen = fn()
  
  function next(err, data){
    let result = gen.next(data)
    if(result.done) return
    result.value(next)
  }
  
  next()
}

function* g(){
  let f1 = yield readFileThunk('fileA')
  let f2 = yield readFileThunk('fileB')
}

run(g)
```

有了这个执行器，执行Generator函数就方便多了。不管内部有多少个异步操作，直接把Generator函数传入`run`函数即可。

当前，前提是每一个异步操作，都要是Thunk函数，即跟在yield命令后面的必须是Thunk函数。

#### co模块

[co模块](https://github.com/tj/co)是著名程序员TJ Holowaychuk于2013年6月发布的一个小工具，用于Generator函数的自动执行。

co模块可以让你不用编写Generator函数的执行器，如下所示：

```javascript
let gen = function* (){
  let f1 = yield readFile('/etc/fstab')
  let f2 = yield readFile('/etc/shells')
  console.log(f1.toString())
  console.log(f2.toString())
}

const co = require('co')
co(gen)
```

`co`函数返回一个Promise对象，因此可以用`then`方法添加回调函数。下面代码中，等到Generator函数执行结束，就会输出一行提示：

```javascript
co(gen).then(() => {
  console.log('Generator函数执行完成')
})
```

## 4. 异步方案优化：async和await

前文提到过，thunkify模块和co模块解决了Generator函数自动执行的问题。等到ES2017标准引入`async`函数后，异步操作就变得更加方便了。

`async`函数是什么？如果用一句话来概括，它就是Generator函数的语法糖。

我们来看实际例子，前文有一个Generator函数，依次读取两个文件：

```javascript
const fs = require('fs')

/* 相当于将fs.readFile转换为Thunk函数 */
const readFile = function(filename){
  return new Promise((resolve, reject) => {
    fs.readFile(fileName, (error, data) => {
      if(error) return reject(error)
      resolve(data)
    })
  })
}

const gen = function* (){
  const f1 = yield readFile('/etc/fstab')
  const f2 = yield readFile('/etc/shells')
  console.log(f1.toString())
  console.log(f2.toString())
}
```

上面代码中的函数`gen`可以写成`async`函数：

```javascript
const asyncReadFile = async function(){
  const f1 = await readFile('/etc/fstab')
  const f2 = await readFile('/etc/shells')
  console.log(f1.toString())
  console.log(f2.toString())
}
```

怎么样，是不是简洁了很多？代码看起来也更容易阅读了？

把这两个例子一对比，就会发现：`async`函数就是将Generator函数的星号(`*`)替换成了`async`，将`yield`替换为`await`。

`async`函数对Generator函数的改进，体现在以下四点：

1. 内置执行器：Generator函数的执行必须靠执行器，所以才有了`co`模块。而`async`函数自带执行器，执行与普通函数一模一样。
2. 更好的语义：`async`表示函数里有异步操作，`await`表示此处需要等待结果。
3. 更广的适用性：`co`模块约定，`yield`命令后面只能是Thunk函数或Promise对象。而`async`函数的`await`命令后，可以是Promise对象和原始类型的值(会自动转成立即`resolved`的Promise对象)。
4. 返回值是Promise：`async`函数的返回值是Promise对象，这比Generator函数的返回值是Iterator对象更加方便。

### 一些常见的异步场景

#### 1. 按顺序完成异步操作

实际开发中，经常遇到一组异步操作，需要按顺序完成。比如，依次远程读取一组URL，然后按读取的顺序（http请求返回的顺序）输出结果。

```javascript
/*
* 注意：async函数始终返回一个Promise对象，即使函数内部没有return语句
*/
async function logInOrder(urls){
  const textPromises = urls.map(async url => {
    const response = await fetch(url)
    return response.text()
  })
  
  for(const textPromise of textPromises){
    const text = await textPromise
    console.log(text)
  }
}
```

上面代码中，虽然`map`方法的参数是`async`函数，但它是并发执行的，因为只有`async`函数内部是继发执行，外部不受影响。后面的`for...of`循环内部使用了`await`，因此实现了按顺序输出。

题外话：`map`并发执行，与`for...of`继发执行的区别：

```javascript
/*
* map 并发执行：
*/
function resolveAfterNSeconds(url) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(url + 'resolved');
    }, 1000 * url);
  });
}

async function concurrentAsyncCall(urls) {
  urls.map(async url => {
    console.log(url)
    const res = await resolveAfterNSeconds(url)
    console.log(res)
  })
}

concurrentAsyncCall([5,4,3,2,1])

// output：
// 5,4,3,2,1,1resolved，2resolved，3resolved，4resolved，5resolved

/*
* for...of 继发执行：
*/
async function secondaryAsyncCall(urls) {
  for(const url of urls){
    console.log(url)
    const res = await resolveAfterNSeconds(url)
    console.log(res)
  }
}

secondaryAsyncCall([5,4,3,2,1])

// output
// 5,5resolved, 4,4resolved, 3,3resolved, 2,2resolved, 1,1resolved
```

### 2. Promise串行继发执行

这个场景源于工作实际开发中，需要逐个串行向后端发送数目不定的Promise请求。即：第一个Promise请求成功后，才会发送第二个Promise请求；也就是说，继发执行。

所以无论是`Promise.all()`，还是`Promise.race()`都不适用。

经过查阅资料，最终得到了两种实现方法：`Array.reduce`和`for...of`：

```javascript
/*
* getData：表示一个request请求
* 当然，此处用setTimeout来模拟一下
*/
function getData(url){
  return new Promise(resolve => {
    const time = Number.parseInt(Math.random() * 10000)
    setTimeout(() => {
      resolve({time})
    }, time)
  })
}

/*
* Array.reduce
* 此处userIds代指数量不定的Promise请求
* 参数time并没有具体的意义，只是代表可能存在的参数
*/

let urls = [
    'https://www.baidu.com/',
    'https://cn.bing.com/',
    'https://www.google.com/'
]

urls.reduce(async (previousPromise, url) => {
    const time = await accumulatorPromise
    return new Promise(resolve => {
        getData(url).then(res => {
            resolve(res.time)
        })
    })
}, Promise.resolve({time: 0}))

/*
* for...of：语义性更好
* 因为for...of会自动去寻找迭代器接口
* 所以可以保证操作的继发性，
*/
async function serialPromise(urls){
  for(const url of urls){
    const res = await getData(url)
    console.log(res.time)
  }
}
```

### 一些注意事项

[1] `await`命令后面的`Promise`对象，运行结果可能是`rejected`，所以最好把`await`命令放在`try...catch`代码块中。如果不作处理的话，最终该错误会被`async`函数返回的`Promise`对象的`catch`捕获。

[2] 多个`await`命令后面的异步操作，如果不存在继发关系，最好让它们同时触发。

```javascript
let [foo, bar] = await Promise.all([getFoo(), getBar()])

/* 另一种写法 */
let fooPromise = getFoo()
let barPromise = getBar()
let foo = await fooPromise
let bar = await barPromise
```

[3] `await`命令只能用在`async`函数之中，如果用在普通函数中，就会报错。

## 参考资料

1. [Iterator（遍历器）的概念 - ES6标准入门教程 - 阮一峰](https://www.bookstack.cn/read/es6-3rd/spilt.1.docs-iterator.md)
2. [Generator 函数的语法 - ES6标准教程 - 阮一峰](https://www.bookstack.cn/read/es6-3rd/docs-generator.md)
3. [async 函数 - ES6标准教程 - 阮一峰](https://www.bookstack.cn/read/es6-3rd/docs-async.md)

