---
title: vdom深入分析
subtitle: Vue技术系列
author: 傅腾达
link: https://juejin.im/post/5cad9fc96fb9a068a75d2909
---

## 概览
1.  遇到的关于key的问题
2.  VDOM深入分析

## 遇到的问题

在使用Vue渲染“可删减”的列表时，错误的使用index作为key，导致列表视图出现错乱。

[点击查看问题](https://codepen.io/fxxjdedd-the-encoder/pen/qwrdVe)

这个问题一下子很难解释，下面我们通过几个小问题，一步一步来分析。

## 

为什么会触发组件的update：

[查看使用index作为key例子](https://codepen.io/fxxjdedd-the-encoder/pen/WWpvdX)

测试：删除第一行，查看控制台打印情况

##

DOM的实际行为：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05734b058ac7c?w=454&h=422&f=png&s=26638)

删除第一行这句话本质上其实是：删除vue实例数据中list的第一项，并不是删除dom的第一个节点！

## 

VDOM的diff算法：

不要被吓到，其实就是4个cursor + 4个node + 4个辅助变量

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05e415aba722f?w=737&h=376&f=png&s=52890)

## 

在我们的例子里将会是这样：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05e619a990c2e?w=476&h=327&f=png&s=20189)

## 

继续往下看：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05e7c6d79ce0f?w=717&h=342&f=png&s=50335)

## 

如何判断是否是sameVnode：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05f660f5d14f2?w=454&h=309&f=png&s=28562)

## 

可以解释为什么会update了：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a05e619a990c2e?w=476&h=327&f=png&s=20189)

![](https://user-gold-cdn.xitu.io/2019/4/10/16a072350da8a624?w=757&h=85&f=png&s=16264)

## 

patchVnode做了什么：

就做了一件事：re-render

cat之所以变成dog，不是因为新建了一个dog节点，而是cat节点被复用，然后使用新的props，通过re-render实现了视图的更新！

## 

回到开始的问题：

如下图，到这里我们应该已经理解为什么删除第一行后，cat会变成dog。但是，为什么`<input />`里的1没有变成2呢？

![](https://user-gold-cdn.xitu.io/2019/4/10/16a06195202d1619?w=906&h=120&f=png&s=13533)

## 

一个简单的解释：

我们之前说过：`patchVnode`的结果，其实就是使用新的props，让这个cat节点进行re-render。

这里是re-render，它的执行不是unmount一个节点，然后再mount一个新的节点，而是直接使用新的props来receive（更新）一个节点，节点的instance并没有重置，所以re-render的过程中，data压根就没变。

## 这就结束了吗？

在解决一个问题的时候，通过阅读源码来找到问题的答案固然是没错的，但是如果一直局限于源码之中，反而不利于我们对框架的进一步理解。

## VDOM深入分析

怎么理解框架设计的意图：

其实我们一直在讨论的就是“如何更新”这件事，要理解这里的意图，我们就要站在框架设计的角度去思考，如何实现一个vdom到dom的更新系统。

## 
什么是VDOM：

VDOM是虚拟DOM，是一种理论，一种解决声明式编程高效更新DOM的方案。

要实现一个vdom到dom的更新系统，需要先基于vdom实现一个：

统一"user-defined"组件和"host元素"的抽象，这个抽象就是组件化。

##
基于组件化的抽象：

![](https://user-gold-cdn.xitu.io/2019/4/11/16a0a68b358c371d?w=789&h=551&f=png&s=47079)

## 
到目前为止，我们其实完成了vdom部分的工作：

输入数据，输出Element

那么剩余的工作就是把vdom更新到dom上，这其中包含三个部分：

1.  mount
2.  unmount
3.  update

## 

这一步的工作，要考虑到我们的runtime可能运行在不同的平台上，比如web、native、desktop等。


- 在React中，把这个工作交给了不同的渲染器：react-dom、react-native-renderer、react-art
- 在Vue中，没有单独抽离，而是直接根据平台类型，给Vue.prototype上赋予了平台相关性的$mount、_update、__patch__等方法

在这里我们以react-dom为例。

##

如何把element给mount到dom上：

![](https://user-gold-cdn.xitu.io/2019/4/11/16a0b9140d46c574?w=1060&h=610&f=png&s=110396)

##

如何unmount：

unmount要做两件事：

1.  在vdom层面上，需要触发componentWillUnmount生命周期钩子
2.  在dom层面上，需要从dom树上移除当前节点

## 

![](https://user-gold-cdn.xitu.io/2019/4/11/16a0b42c5f862b27?w=1142&h=439&f=png&s=69352)

##

如何在数据变化后update视图：

update就是使用新的props重新渲染视图，其中有两种方案:

1.  直接umount整个树，然后用新props重新mount整个树
2.  复用已经mount的树，根据diff结果动态的修改已经存在的dom树

我们肯定采用的第二种方案。

## 

一个需要注意的地方：

我们要知道，如果只有Element

1. 阿斯顿
2. diff的时候，我们必须重新render一次，才能进行对比，否则你根本不知道这个组件长什么样
3. setState做了什么
4. render做了什么

## note

？？？

组件是在什么时候实例化的？
实在react-dom里实例化的吗？
组件又是如何反映到DOM上的？

key可以重置的原因


<style type="text/css">
@import "../align.css";
</style>