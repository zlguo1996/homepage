---
title: React 18前瞻
categories:
  - code
  - frontend
tags:
  - react
---



## 简介

React 18 的所有新特性基本可以归为下面两类：

1. 并发渲染。为了解决大量DOM节点同时更新造成的渲染延迟问题，以及子组件IO操作被父组件阻塞的问题。React新的Fiber Reconciler通过将任务拆分成小块，支持了并发渲染。既：允许React同时渲染多版本的UI。
2. 新SSR架构。允许用户将应用拆分为更小的单元，使每个单元具备独立的fetch data (server) → render to HTML (server) → load code (client) → hydrate (client)流程。

## 新特性

### 并发渲染

在更早的时候，react提出了实验性的特性[concurrent mode](https://reactjs.org/docs/concurrent-mode-intro.html)。通过concurrent modes实现了并发的状态更新，这里的并发有两种内含：

1. 对于CPU相关的更新（如创建DOM节点，执行组件生命周期函数），这意味着高优的更新能够打断正在执行的更新。
2. 对于IO相关的更新（如请求数据），这意味着在数据返回前可以在内存中提前进行渲染。

在react 18的[post](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html)中，作者将其重新命名为"concurrent rendering"。意味着react18中并发不再是以一种必须选择的作用于全局的模式（all-or-nothing “mode”），而是只会在需要时被新特性触发的特性。这也和react18的渐进式升级策略相关联。

#### CPU-bounded updates

为了提升运行性能和用户体验，react修改渲染的调度机制，这主要包含两点：

1. 将一次渲染分片，允许多次渲染并发执行，也支持渲染的打断
2. 对渲染区分优先级，优先执行高优渲染任务

##### 分片

这个能力再react 16引入的Fiber Reconciler中就已实现。在此之前react使用的是Stack Reconciler，它使用递归的方式对整棵树进行渲染

stack reconciler & fiber reconciler https://zhuanlan.zhihu.com/p/109971435 TODO

##### 优先级

react18将状态更新分为两个类别（优先级）：

1. Urgent updates：需要快速反馈的交互，如：键盘输入、点击、触摸等
2. Transition updates：UI从一个视图到另一个视图的转换

比如：在一个cms的表格场景中，当用户选中一个下拉框的选项来过滤列表的时候。用户期望在点击选项后下拉框快速收起并更新选中项（urgent updates），但是真实的过滤后的列表无需即时变化（transition updates）。

在18之前，所有的状态更新都是urgent updates，但实际上很多场景在18中可以归类为transition updates。但为了向后兼容，transition updates被作为一个可选的特性，只有当用户使用对应的API触法状态更新的时候才会被使用。

**API**

这个新的API就是`startTransition`：

```js
import { useTransition } from 'react';


const [isPending, startTransition] = useTransition();

// Mark any state updates inside as transitions
startTransition(() => {
  setState(input);
});
```

useTransition hook 返回了：

1. isPending：表示当前的状态更新还未被反映（渲染）到视图上
2. startTransition：用于触发transition updates
   - startTransition会被同步执行，但是这个更新会被标记，在更新被处理时react会以此判断如何渲染更新
   - 渲染时，startTransition造成的更新是可以被打断的，它不会block页面。当用户的输入改变后，react将不会继续渲染过期的更新

**应用场景**

在以下的场景中，我们可以用`startTransition`API来替代之前的状态更新API：

1. Slow rendering：当一个更新需要消耗大量计算资源的时候
2. Slow network：当一个更新需要react等待接口返回数据的时候

可以这么认为：当一个更新本身就需要耗费一些等待时间（等待可能是来源于大量计算的开销或网络的等待）时，那么用户不会在乎为这个更新再多等待一些时间，此时就可以使用startTransition API。以此，可以将计算资源更多地提供给urgent updates来提升用户体验。

> react18和之前版本的性能比较可以见 [这个案例](https://github.com/reactwg/react-18/discussions/65)

#### IO-bounded updates



#### Break Change

部分生命周期的执行时机和次数将会发生改变。我们可以将react的生命周期分为两类：render phase（渲染阶段）和commit phase（提交阶段），详见[react-lifecycle-methods-diagram](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)。在并发渲染时，一个组件的更新可能会被打断，也有可能会被重新恢复，这就导致了一个渲染阶段的生命周期可能会被多次执行，造成切换到并发渲染后组件发生预期外的表现。因此在将旧组件升级为并发渲染时，需要注意：

1. 将render phase生命周期回调放到commit phase的回调中执行。
2. 或保证render phase执行的逻辑是[幂等](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning)的。即该回调中的side effect，**多次执行**与**单次执行**对系统状态的影响**相同**。

在开发阶段，我们可以：

1. 通过React的[Strict Mode](https://reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects)来检测这些潜在的错误（Strict Mode并非直接检测副作用，而是将这些生命周期的回调执行两次以便于用户发现非幂等的副作用）
2. 不再使用componentWillMount等生命周期，这些生命周期的替代方案可以参考[官方文档](https://reactjs.org/docs/react-component.html#legacy-lifecycle-methods)（这也是为什么React16.9将这些函数命名为UNSAFE_componentWillMount等，并在控制台打印警告）

### createRoot

### automatic batching

### startTransition

### new streaming server renderer

## 渐进式升级

仅管

## 里程碑

- 2021-06-08 发布alpha包
- 2021-11-15 发布beta包

## References

- [The Plan for React 18](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html)
- [React 18 就要来了，来看看发布计划 🤩](https://zhuanlan.zhihu.com/p/379072979)
- [理解 React Fiber & Concurrent Mode](https://zhuanlan.zhihu.com/p/109971435)
- [Introducing Concurrent Mode (Experimental)](https://reactjs.org/docs/concurrent-mode-intro.html)
- [Legacy Lifecycle Methods](https://reactjs.org/docs/react-component.html#legacy-lifecycle-methods)
