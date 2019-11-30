# 通过几个问题深入浅出Vue

## 🎙 前言
通常，Vue给我们的印象是“小巧易用”，凭借其简洁明了的模板开发方式，以及强大的指令系统，我们可以轻轻松松几行代码搞定一个数据双向绑定的页面。但是，这背后Vue帮我们做了多少工作，我们是知之甚少的。

Vue就像一个黑盒子，我们输入一些数据，它给我们输出一个渲染好的页面。对于开发，这很方便，我们不需要关心何时去触发更新，因为Vue已经帮我们做了。但是，在面对一些棘手问题时，我们需要去分析数据是何时变化的、被谁更改的、为什么会触发更新、为什么又不能触发更新，等等问题，这个时候就很难，因为这些逻辑都隐藏在黑盒子内部，我们无法观察，更无法控制。

所以说，想要用好Vue其实还挺难的。

面对这些问题，我们往往会基于个人的经验，个人的理解去分析问题，会花费大量时间去debug，这很低效。不如我们先抽点时间，了解一下Vue的实现细节吧。

## 🔑问题分类
其实按照Vue的开发方式，一般都会有如下流程：

  1.  先初始化一个Vue实例，然后传入各种配置信息
  2.  尝试修改一些数据
  3.  期待视图更新

所以，我们遇到的问题可以归为以下三类：

  1.  Vue的初始化流程是怎样的
  2.  数据的更改会不会触发视图的更新
  3.  数据的更改何时会触发视图的更新



## 💡问题
## Q1. 我对Vue的配置信息不理解（第一类问题）
通常我们的一个Vue实例是这样的:
```js
const options = {
  props: ...,
  data: ...,
  computed: ...,
  watch: ...,
  methods: ...,
  created: ...,
  mounted: ...
}
new Vue(options).$mount('#app')
```
也许，我们很清楚每个属性的含义，但是我们却不知道这些属性是如何被调用的，是以怎样的顺序来初始化的。通过这种”无顺序“配置对象的方式来构建Vue实例，我们先天性的丢失了一些重要的“信息”，那就是代码的执行顺序，这会带来一系列理解上的问题。

其实，通过文档里的生命周期图，或者看源码，我们就会知道，在`new Vue(opions)`的这个过程中，即Vue这个类的构造函数中，Vue会依次”同步“地把`props,methods,data,computed,watch`取出来，然后分别初始化，之后再执行我们的created方法。最后, 在我们执行`$mount('#app)`之后，再执行我们的mounted方法。

知道了这个顺序，就能解决很多疑惑比如：

1.  在watch, created, mounted里面第一次用计算属性的时候，计算属性已经初始化了吗？
2.  data属性能否使用计算属性来初始化
3.  在created里修改watch监听的属性，会不会触发watch的执行？如果加了immediate又会执行几次呢？
4.  ...等等自己不确定的问题

## Q2. 我修改数据为什么没有生效（第二类问题）
我们或多或少都会遇到，明明自己改了数据，可是视图就是不更新的问题，举个例子：
```xml
<template>
  <div>
    <p>{{lib}}</p>
    <p>{{detail.version}}</p>
    <p>{{detail.type}}</p>
  </div>
</template>
```
```js
export default {
  data() {
    return {
      lib: 'vue',
      detail: {
        version: '2.5.1',
        // type: 'fe'
      }
    }
  },
  mounted() {
    this.detail.type = 'fe'
  }
}
```

```
// 输出
vue
2.5.1
```
问题很容易看出，是因为我们没有事先在data里声明好type这个属性，所以在data的初始化过程中，并没有observe（即把属性转变为getter和setter）这个type属性，所以我们的修改是不会触发setter的，也就不会引起视图更新。

接下来，我们更新一下代码:
```js
// 把mounted改成created
created() {
  this.detail.type = 'fe'
}
```
我们神奇的发现，竟然显示出来了，这跟我们刚才的结论相悖呀。其实，我们的结论没错，之所以这次能显示出来，纯粹是巧合。我们知道created是在data初始化之后调用的，同时又是在mounted之前调用的，所以data在mounted之前就被赋予了一个未被observe的属性type，然后在$mount的时候，顺带显示在页面上了。也就是说，这次的显示，并非setter的触发，而是本来data就已经有了type属性罢了。

我们再来改一下:
```js
created() {
  this.detail.type = 'fe'
},
mounted() {
  this.detail.type = 'be'
}
```
我们会发现，依旧显示的是fe，而不是be，这验证了上一个结论。另外，一定要好好理解官网的这个lifecycle图: 

<img src="images/lifecycle.png" width="450">

**一句话：只有被observe的数据改变后才会触发视图的更新**

## Q3：Vue是如何使用事件循环的（第三类问题）
这个问题比较抽象，具体一点举几个例子：

  -  怎么准确的判断watch的handler什么时候执行、执行几次
  -  我更改了数据，何时才会触发视图更新
  -  $nextTick和setTimeou区别

可以说，**Vue的本质其实就是一套精心设计的事件循环系统**，要弄懂Vue必须弄懂两件事：

  1.  事件循环本身
  2.  Vue的事件循环系统

事件循环需要理解到task和microTask的层次，而Vue的事件循环系统需要读透文档，理解作者的思想，另外多看源码。下面我们举一些例子

### Q3-1：watch的handler什么时候执行

```js
export default {
  data() {
    return {
      lib: 'vue',
      lock: true
    }
  },
  watch: {
    lib(val, old) {
      if(this.lock) {
        console.log(`lib changed from ${old} to ${val}`)        
      }
    }
  },
  created() {
    this.lock = false;
    this.lib = 'react'
    this.lock = true;
  }
}
```

```js
// 输出
lib changed from vue to react
```
按照我们常规的理解，当我们执行`this.lib = 'react`的时候，理应当触发watch，而此时lock其实是false，不应该输出。这里有一个『陷阱』，就是我们错误的以为Vue的内部的执行机制是同步的，而事实上，Vue会充分利用事件循环做一些异步的事情，比如这里的handler执行机制。

根据Q1，并结合一定的源码阅读，我们知道本次示例的初始化顺序为：

  1. 先初始化data，取出lib、lock两个属性，分别observe
  2. 初始化watch，取出lib属性的key和value(即handler)，并用他们初始化一个watcher，从而形成一个watcher对lib属性的监听，并等待一个『合适的时机』去执行我们给定的handler
  3. 执行created声明周期函数，此时数据的初始化工作已经完成，接下来先执行`this.lock = false`，这不会影响什么。再执行`this.lib = 'react'`, 此时触发了lib的setter方法，接着Vue会找出所以正在监听lib属性的watcher，并执行其update方法
```js
update () {
  queueWatcher(this)
}
```
  我们发现，watcher并没有立即执行handler，而是发起了一个queueWatcher：

```js
flushing = false
waiting = false
export function queueWatcher (watcher) {
  if (!flushing) {
    queue.push(watcher)
  }
  // queue the flush
  if (!waiting) {
    waiting = true
    // flushSchedulerQueueh会依次把queue中的watcher拿出来执行
    nextTick(flushSchedulerQueue) 
  }
}
```
已知flushing和wating默认值都是false，所以『第一次触发watch』代码会像这样执行
```js
queue.push(watcher)
nextTick(flushSchedulerQueue)
```
可以看到，我们的handler会在nextTick时执行（关于nextTick我们后面会讲，在这里可以暂时理解成setTimeout），而等到下一次事件循环，`this.lock = true`已经执行，所以我们console了出来。

总结：我们现在l对Vue的事件循环机制有了一个认知，即Vue中数据的变化所引起的响应，是依托事件循环就机制来完成的。

### Q3-2: 深入Vue的事件循环细节
仅知道数据的响应是依托于事件循环还不够，因为我们的代码会越写越复杂，常常会有多个异步任务，此时我们需要准确的知道，我们的watch何时触发，我们的视图何时更新。因此，我们需要更加细致的去研究。

#### Q3-2-1：直接修改值

```js
export default {
  data() {
    return {
      lib: 'vue'
    }
  },
  watch: {
    lib(val, old) {
      console.log(`lib changed from ${old} to ${val}`)        
    }
  },
  created() {
    this.lib = 'react'
    this.lib = 'angular'
  }
}
```
```js
// 输出
lib changed from vue to angular
```
问题有两个：

  1. 为什么watch只执行一次 
  2. 为什么是angular

虽然这个问题比较蠢，但是为了帮助我们理解后面的例子，我们还是得深入的去研究一下。

根据Q3-1，watch中的lib会创建一个watcher，并监听lib属性的变化。当我们第一次`this.lib = 'react'`, 此时会触发lib的setter，并找到正在监听的watcher，依次执行watcher的update方法，update方法会调用queueWatcher方法，现在我们看一下queueWatcher更完整一点的代码：

```js
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  // has维持着所有watcher的id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      ...
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```
可以看到，这里有一个`if (has[id] == null)`的判断，要知道，我们watch只有一个lib属性，所以只会初始化一个watcher，所以当第二次执行`this.lib = 'angular'`的时候，queueWatcher其实什么都没干。

所以结果是，虽然有两次lib的变化，但是watcher会在下一次事件循环，只执行一次handler。


#### Q3-2-2 在setTimeout里修改值
```js
export default {
  data() {
    return {
      lib: 'vue'
    }
  },
  watch: {
    lib(val, old) {
      console.log(`lib changed from ${old} to ${val}`)        
    }
  },
  created() {
    setTimeout(() => {
      this.lib = 'react'
    }, 0)
    setTimeout(() => {
      this.lib = 'angular'
    }, 0)
  }
}
```
```js
// 输出
lib changed from vue to react
lib changed from react to angular
```
为什么现在又输出两次了？再看一遍queueWatcher方法：
```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  // has维持着所有watcher的id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      ...
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```
还需要看flushSchedulerQueue方法
```js
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  flushing = true
  let watcher, id
  ...
  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    ...
  }
  ...
}
```
[点我查看预备知识（事件循环中的task和microTask）](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

当Vue执行created方法，会执行两个setTimeout，而setTimeout又是task，所以我们写的两个setTimeout函数，会分别在两次事件循环中执行，而不是一次，如下：

1.  第一次事件循环(script任务, 也是task)：vue初始化，created方法发起两个异步任务
2.  第二次事件循环(task)：开始执行第一个setTimeout的callback
3.  第三次事件循环(task)：开始执行第二个setTimeout的callback

在第二次事件循环中，我们修改了lib，同时出发了相应的watcher，最终执行queueWatcher方法，于是当前的watcher被push到queue队列，待到nextTick执行，而nextTick是microTask，于是我们上面的过程变成了：

1.  第一次事件循环(script任务)：vue初始化，created方法发起两个异步任务
2.  第二次事件循环(task)：开始执行第一个setTimeout的callback
3.  第二次事件循环(microTask)：`nextTick(flushSchedulerQueue)`中执行watcher.run(即执行handler)，并清空has[id]
4.  第三次事件循环(task)：开始执行第二个setTimeout的callback

注意：在js的一次事件循环中，先执行所有同步代码，之后，会从macroTask队列里取出1个macroTask执行。然后，再取出所有microTask队列里的microTask，并依次执行。这整个过程结束后，便会开启下一次事件循环。

接下来到了第三次事件循环，我们再次修改lib，同样的过程，因为上一步已经清空了has[id]，所以本次lib 的更新其实跟上一次一模一样，所以过程变成了：

1.  第一次事件循环(script任务)：vue初始化，created方法发起两个异步任务
2.  第二次事件循环(task)：开始执行第一个setTimeout的callback
3.  第二次事件循环(microTask)：`nextTick(flushSchedulerQueue)`中执行watcher.run(即执行handler)，并清空has[id]
4.  第三次事件循环(task)：开始执行第二个setTimeout的callback
5.  第三次事件循环(microTask)：`nextTick(flushSchedulerQueue)`中执行watcher.run(即执行handler)，并清空has[id]

所以，会打印两次。

#### Q3-2-3 在nextTick里修改值

```js
export default {
  data() {
    return {
      lib: 'vue'
    }
  },
  watch: {
    lib(val, old) {
      console.log(`lib changed from ${old} to ${val}`)        
    }
  },
  created() {
    this.$nextTick(() => {
      this.lib = 'react'
    })
    this.$nextTick(() => {
      this.lib = 'angular'
    })
  }
}
```
问题是，为什么watch只执行了一次。

1.  第一次事件循环(script任务)：vue初始化，created方法发起两个microTask异步任务
2.  第二次事件循环(task)：没有macroTask
3.  第二次事件循环(microTask)：现在microTask任务队列是这样的`[callback1, callback2]`, 从microTask队列取出第一个callback1，执行，触发lib更新，从而执行queueWatcher，在里面又触发了一个nextTick(microTask)异步任务，并push到microTask任务队列，队列变成这样`[callback2, callback3]`
4.  第二次事件循环(microTask)：从micoTask队列取出callback2，执行，触发lib更新，从而执行queueWatcher，而此时has[id]已经有值(因为callback3还没执行，因此has[id]还没被清空)，所以直接略过
5.  第二次事件循环(microTask)：从microTask队列取出callback3，执行

可以看到，这就是只打印一次的原因了。

## 总结

其实Vue的设计思想就是：事件循环+双向绑定，只要我们搞明白这两点，我们就可以真正的掌握Vue，写出稳定、可预测的代码，轻松的解决使用中遇到的各种问题。

有了设计思想还不够，还需要工具去实现这种思想，那就是compiler和vdom干的事了，还有很多东西要学呢💪🏻。

