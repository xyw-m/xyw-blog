---
title: Vue中的$nextTick有什么作用？
author: xyw
date: 2023-04-03 10:43:43
comments: true
tags:
- Vue
categories:
- 前端
---

首先明确一个事实：Vue在更新DOM时是异步执行的。Vue官网在[深入响应式原理](https://v2.cn.vuejs.org/v2/guide/reactivity.html)一节中是这么描述异步更新队列的：

> 当侦听到数据变化，Vue将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个watcher被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和DOM操作是非常重要的。
>
> 然后，在下一个事件循环"tick"中，Vue刷新队列并执行实际(已去重的)工作。
>
> Vue在内部对异步队列尝试使用原生的`Promise.then`、`MutationObserver`和`setImmediate`，如果执行环境不支持，则会采用`setTimeout(fn, 0)`代替。

<!--more-->

也就是说，当我们设置`this.msg = 'hello world'`的时候，DOM节点的文本(可以简单的理解为`Node.textContent`)并没有立即更新，而是将这个操作放进一个队列里。如果我们重复执行的话，队列还会进行去重操作。等到同一事件循环中的所有数据完成变化之后，再逐一处理队列中的事件。

那么如果我们想在上述的赋值操作后立即操作DOM呢？比如：

```javascript
export default {
  methods: {
    this.msg = 'hello world'
    console.log(this.$el.textContent)
  }
}
```

这时候就需要用到`Vue.nextTick(callback)`了。

## 1.$nextTick是什么？

先来看一下Vue官方对nextTick的定义(此处摘录的是Vue3官方文档的定义)：

> 当你在Vue中更改响应式状态时，最终的DOM更新并不是同步生效的，而是由Vue将它们缓存在一个队列中，直到下一次"tick"才一起执行。这样使为了确保每个组件无论发生多少状态改变，都仅执行一次更新。
>
> 
>
> `nextTick()`可以在状态改变后立即使用，以等待DOM更新完成。

如上文所述，修改响应式数据后，Vue将开启一个异步更新队列，视图需要等待队列中所有数据变化完成后，再统一进行更新。

即只有在`nextTick()`的回调函数中，才能稳定获取到更新后的DOM结构。

## 2.如何使用$nextTick

该函数有两个参数，第一个参数是回调函数，第二个参数是执行函数上下文。

```javascript
// 修改数据
vm.message = '修改后的值'
// 此时DOM还未更新
console.log(vm.$el.textContent)	// 原始值
Vue.nextTick(function(){
  // DOM已更新
  console.log(vm.$el.textContent)	// 修改后的值
})
```

组件内使用`vm.$nextTick()`实例方法只需要通过`this.$nextTick()`，并且回调函数中的`this`将自动绑定到当前的Vue实例上。

此外，`$nextTick()`会返回一个`Promise`对象，所以也可以用`async/await`。

```javascript
this.message = '修改后的值'
console.log(this.$el.textContent)	// 原始值
await this.$nextTick()
console.log(this.$el.textContent)	// 修改后的值
```

## 3.进阶：实现原理

我们可以根据nextTick的源码分析该函数的实现原理，源码位于：[/src/core/util/next-tick.ts](https://github.com/vuejs/vue/blob/main/src/core/util/next-tick.ts)

首先是nextTick部分(省略了一些不必要的代码)：

```javascript
export let isUsingMicroTask = false

const callbacks:Array<Function> = []		// 回调函数队列
let pending = false		// 用来标识同一个时间内只能执行一次

export function nextTick(cb?:(...args: any[]) => any, ctx?: Object){
  let _resolve
  
  // 调用nextTick会将传入的cb回调函数(绑定执行上下文后)存入callbacks数组
  callbacks.push(() => {
    if(cb){
      try {
        cb.call(ctx)
      } catch(e:any){
        handleError(e, ctx, 'nextTick')
      }
    } else if(_resolve){
      _resolve(ctx)
    }
  })
  
  // 执行异步延迟函数 timerFunc
  if(!pending){
    pending = true
    timerFunc()
  }
  
  // 当nextTick没有传入回调函数时，返回promise
  if(!cb && typeof Promise !== 'undefined'){
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

然后是timerFunc部分：`timerFunc`只是将`flushCallBacks`添加进微任务队列(或宏任务队列)，同一事件循环内只会执行一次。

`timerFunc`会根据当前环境支持什么异步方法来确定调用哪个。优先级分别为：`Promise.then` > `MutationObserver` > `setImmediate` > `setTimeout`。

此处贴一段timerFunc的注释(个人翻译版)：

> 此处我们使用微任务(microtasks)进行异步延迟封装。在2.5，我们本来使用了(宏)任务结合微任务的方式，然而，当状态(这里应该指的是响应式数据)正好在重绘前改变时，会产生一些微妙的问题，例如\#6813, out-in transitions。
>
> 此外，在事件处理器中使用(宏)任务将导致一些无法回避的奇怪行为，例如\#7109, #7153, #7546, #7834, #8109。
>
> 因此，我们现在到处使用微任务。
>
> 这种权衡带来的一个主要的缺点是：在某些场景下，微任务的优先级太高，并在所谓的连续事件之间(例如#4521，#6690，它们有变通方法)，甚至在同一事件冒泡之间触发(#6566)。
>
> 
>
> nextTick行为利用了可访问的微任务队列，原生Promise.then和MutationObserver都可以被添加到为任务队列。MutationObserver有更广泛的支持，但是它在UIWebView中有严重的漏洞：在iOS >= 9.3.3时触发触摸事件处理程序。它在触发几次后完全停止工作....所以只要原生Promise可用，我们就会使用它。
>

```javascript
let timerFunc

if(typeof Promise !== 'undefined' && isNative(Promise)){
  // Promise.resolve()相当于直接返回了一个已经resolve的promise对象
  // 此处的操作只是为了将flushCallBacks添加进微任务队列
  // flushCallBacks将会依次调用队列中的回调函数，详见下文
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if(isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if(
  !isIE &&
 	typeof MutationObserver !== 'undefined' &&
  (isNative(MutationObserver) || 
  MutationObserver.toString() === '[object MutationObserverConstructor]')
){
  // Use MutationObserver where native Promise is not available
  // MutationObserver 会在指定的DOM发生变化时被调用
  // 此处创建了一个文本节点并使用MutationObserver来监视它
  // 然后手动更改文本，从而触发MutationObserver回调函数
  // 这些操作也是为了利用MutationObserver将flushCallbacks添加进微任务队列
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {characterData: true})
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} esle if(typeof setImmediate !== 'undefined' && isNative(setImmediate)){
  // 当前环境不支持原生Promise和MutationObserver，也就不能利用微任务队列
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // 当前环境不支持原生Promise和MutationObserver，也就不能利用微任务队列
  timerFunc = () =>{
    setTimeout(flushCallbacks, 0)
  }
}
```

最后是flushCallbacks：所有回调函数都会在`flushCallbacks()`中被依次调用，然后`callbacks`置空，`pending`重置为false。

```javascript
function flushCallbacks(){
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for(let i = 0; i < copies.length; i++){
    copies[i]()
  }
}
```

总结一下：

nextTick内部维护了一个`callbacks`队列、一个`pending`锁、一个`timerFunc`和一个`flushCallbacks`。

- `callbacks`队列：收集当前事件循环内所有的`nextTick`回调，等当前队列的宏任务执行完毕后，在`flushCallbacks`里一次性执行完所有回调函数

- `timerFunc`根据浏览器环境将`flushCallbacks`添加进微任务(当前事件循环内执行)或宏任务(下一轮事件循环执行)

- `pending`：保证不会重复创建`timeFunc`。当`pending`为`false`时，表示当前事件循环内首次将cb添加到`callbacks`队列，此时会创建`timeFunc`，并加锁。后续调用`nextTick`只会往`callbacks`队列添加回调。

## 参考资料

1. [面试官：Vue中的$nextTick有什么作用？](https://github.com/febobo/web-interview/issues/14)
2. [vm.$nextTick - Vue3](https://cn.vuejs.org/api/general.html#nexttick)
3. [Promise.resolve() - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve)
4. [深入响应式原理 - Vue2](https://v2.cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)
5. [Vue中$nextTick源码解析 - 掘金](https://juejin.cn/post/6844904147804749832)

