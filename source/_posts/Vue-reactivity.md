---
title: Vue的响应式原理
date: 2023-06-04 13:50:15
tags:
- Vue
categories:
- 前端
---

首先来探讨一下这个问题，什么是响应性？

响应性这个术语在今天的各种编程讨论中经常出现，Vue3官网在[深入响应式系统](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html)一节中对它的解释如下：

<!--more-->

> 响应性是一种可以使我们声明式地处理变化的编程范式。一个经常被拿来当做典型例子的用例是Excel表格：
>
> |      | A    | B    | C    |
> | ---- | ---- | ---- | ---- |
> | 0    | 1    |      |      |
> | 1    | 2    |      |      |
> | 2    | 3    |      |      |
>
> 这里单元格A2中的值是通过公式`=A0+A1`来定义的，因此最终得到的值为3。如果你试着更改A0或A1，你会注意到A2也随即自动更新了。这就是一种响应性(本文作者注)。
>
> 
>
> 而JavaScript默认并不是这样的，如果我们用JavaScript写类似的逻辑：
>
> ```javascript
> let A0 = 1
> let A1 = 2
> let A2 = A0 + A1
> 
> console.log(A2) // 3
> 
> A0 = 2
> console.log(A2) // 仍然是 3
> ```
>
> 当我们更改A0后，A2不会自动更新。

## 什么是Vue的响应性？

我们都知道，Vue最大的特点就是其低侵入性的响应式系统。那么，Vue的响应式究竟是指什么？或者说，当我们在谈论Vue的响应性时，我们究竟想表达什么意思？

假如我们现在有一个按钮，每次点击它都会实时地显示当前时间。

如果用原生的JavaScript来实现它，我们不得不在每次触发click事件后，手动更新DOM节点的文本内容(`p.textContent = time`)，即使变量`time`已经被更新了。

```html
<body>
  <div>
    <p id="time"></p>
    <button id="btn">更新当前时间</button>
  </div>
<script type="text/javascript">
const btn = document.querySelector('#btn')
const p = document.querySelector('#time')
let time = new Date().toLocaleString()
p.textContent = time
btn.addEventListener('click', () => {
  time = new Date().toLocaleString()
  p.textContent = time
})
</script>
</body>
```

如果用Vue来实现它呢？我们只需更新`time`状态，无需再处理DOM了 ———— Vue会自动帮助我们更新视图。

```vue
<template>
	<div @click="updateTime">
    {{ time }}
  </div>
</template>
<script>
  export default {
    data(){
      return {
        time: new Date().toLocaleString()
      }
    },
    methods: {
      updateTime(){
        this.time = new Date().toLocaleString()
      }
    }
  }
</script>
```

在数据层更新数据，会自动更新视图层。此即Vue的响应式系统。

## 深入响应式原理(Vue2)

(注意，本节中的讲述及示例全部基于Vue2，Vue3实现响应性的方式与Vue2完全不同，后文会单独讨论。)

在实际开发中，我们有时会遇到这样一种情况：为组件的响应式状态(`data`选项)动态添加新属性后，发现这个属性**并不会**响应式更新视图。

例如下例中，定义一个`p`标签，通过`v-for`指令进行遍历。然后给`button`标签绑定点击事件。

```vue
<template>
	<div>
    <p v-for="(value, key) in item" :key="key">
      {{ value }}
  	</p>
    <button @click="addProperty">动态添加新属性</button>
  </div>
</template>
<script>
  export default {
    data(){
      return {
        item: oldProperty: "旧属性"
      }
    },
    methods: {
      addProperty(){
        this.item.newProperty = "新属性"
        console.log(this.item)
      }
    }
  }
</script>
```

我们的预期是：点击按钮时，数据新增一个属性，界面也新增一行。

然而实际的情况是：点击按钮后，数据虽然更新了(`console`打印出了新属性)，但是页面上的视图并没有更新。

为什么item没有响应式更新？想弄清楚原因，首先要了解Vue是如何在JavaScript中实现响应性的。

### Vue2的响应式原理

Vue2通过`Object.defineProperty`来追踪数据变化，从而实现响应式更新。

当你把一个普通的JavaScript对象传入Vue实例作为data选项，Vue将遍历此对象所有的property，并使用`Object.defineProperty`把这些property全部转为[getter/setter](getter/setter)。这些getter/setter对用户来说是不可见的，但是在内部它们让Vue能够追踪依赖，在property被访问和修改时通知变更。

每个组件实例都对应一个watcher实例，它会在组件渲染的过程中把“接触”过的数据property记录为依赖。之后当依赖项的setter触发时，会通知watcher，从而使它关联的组件重新渲染。

![data](https://v2.cn.vuejs.org/images/data.png)

### getter/setter

[Object.definePropery](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)方法用于精确地添加或修改一个对象的属性，这个方法允许修改默认的额外选项。默认情况下，该方法添加的属性值是不可修改(immutable)的。

下例中简单模拟了一下Vue将property转为getter/setter的具体操作：

```javascript
const obj = { val: '' }
Object.defineProperty(obj, 'foo', {
  get(){
    console.log(`get foo: ${this.val}`)
    return this.val
  },
  set(newVal){
    if(newVal !== this.val){
      console.log(`set foo: ${newVal}`)
      this.val = newVal 
    }
  }
})
```

当我们访问`foo`属性，或者为`foo`属性赋值时都能够触发`getter`与`setter`：

```javascript
obj.foo	// get foo 
obj.foo = 'new'		// set foo: new
```

但是我们在为`obj`添加新属性时，却无法触发这种拦截:

```javascript
obj.bar = '新属性'
```

因为在一开始，`obj`的`foo`属性被设成了响应式数据。而`bar`是后面新增的属性，并没有通过`Object.defineProperty`设置成响应式数据。

看到这里，想必你已经明白了为什么item没有响应式更新，因为item的新属性`newProperty`并没有被转为getter/setter,Vue检测不到它的变化，也就无法更新视图。

### 解决办法

那么这种情况有没有解决办法呢？其实是有的。虽然由于JavaScript的限制，Vue不能检测数组和对象的变化。但Vue还是打了些“补丁”来最大程度上保证它们的响应性。

#### 对于对象

1. 对于已经创建的实例，Vue不允许动态添加根级别的响应式property
2. Vue只会在初始化实例时对property执行getter/setter化，所以无法检测property的添加或移除(如上例所示)

但是，如果你想为已有对象添加新的property，可以使用`Vue.set/vm.$set`：

```javascript
this.$set(this.someObject,'b',2)
```

或者如果你需要为已有对象赋值多个新的property，可以这样写：

```javascript
// 代替 `Object.assign(this.someObject, { a: 1, b: 2 })`
this.someObject = Object.assign({}, this.someObject, { a: 1, b: 2 })
```

#### 对于数组

Vue不能检测以下数组的变动：

1. 当你利用索引直接设置一个数组项时，例如：`vm.items[indexOfItem] = newValue`
2. 当你修改数组的长度时，例如：`vm.items.length = newLength`

以下两种方式都可以实现和`vm.items[indexOfItem] = newValue`相同的效果，同时触发响应式更新：

```javascript
// Vue.set
Vue.set(vm.items, indexOfItem, newValue)

// Array.prototype.splice
vm.items.splice(indexOfItem, 1, newValue)
```

如果想要修改数组长度，你同样可以使用数组的`splice`方法：

```javascript
vm.items.splice(newLength)
```

## Vue3的响应式原理

为了实现响应性，Vue需要追踪对象属性的读写。目前在JavaScript中有两种劫持property访问的方式：getter/setter和Proxy。

出于支持旧版本浏览器的限制，Vue2使用了getter/setter。而Vue3中使用了Proxy来创建响应式对象，仅将getter/setter用于ref。

--- 未完待续。

## 参考资料

1. [Vue3 - 响应式原理](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html) 
2. [Vue2 - 响应式原理](https://v2.cn.vuejs.org/v2/guide/reactivity.html) 
