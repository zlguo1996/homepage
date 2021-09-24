---
title: 使用react-use-gesture和react-spring的交互式动画
date: 2021-02-08 15:13:35
categories:
- code
- frontend
- animation
- interaction
---

## 前言

当页面中存在大量诸如页面拖动等动画时，手动处理手势事件，并将其绑定到诸如div等元素的属性将是一件费力且低效的工作。而[react-use-gesture](https://github.com/pmndrs/react-use-gesture)和[react-spring](https://github.com/pmndrs/react-spring)分别对**手势**和**动画**进行了抽象，为使用**react hooks**创建交互式动画提供了一种更为便捷的解决方案。

本文将会先简要介绍使用[react-use-gesture](https://github.com/pmndrs/react-use-gesture)和[react-spring](https://github.com/pmndrs/react-spring)创建交互式动画的方式，以及简要的实现解析。然后描述在开发过程中遇到的问题（兼容性问题&接口问题）及其解决方案。

## react-use-gesture

### 简介

react-use-gesture

1. 对浏览器的用户输入事件进行了封装，提供了统一的手势接口，使复杂的手势（如：拖动，缩放）易于配置。

2. 提供了原生事件所没有的属性（如：速度，距离），丰富了事件所包含的信息。

### 使用

**例子**

一个典型的例子如下所示。使用手势hook中注册回调函数，回调函数负责处理事件并触发side effects。hook会返回一个bind函数，通过调用该函数，将其绑定到react节点上。

```jsx
const bind = useDrag(state => doSomethingWith(state), config)
return <div {...bind(arg)} />
```

>  其中：
>
>  1. 手势hook传入的第一个参数是回调函数，第二个参数是手势配置。
>  2. bind函数中传入参数的方式，常用于给多个节点绑定同一个回调函数，详见[这个例子](https://codesandbox.io/s/fh8r8?file=/src/index.js:1694-1698)。

**手势hook**

目前支持的有普通手势：`useDrag`，`useMove`，`useHover`，`useScroll`，`useWheel`，`usePinch`，以及复合手势：`useGesture`。详细功能见[列表](https://use-gesture.netlify.app/docs/hooks/)。

> 需要区分：
>
> - useDrag，useMove和useHover
>
>   drag事件只有在用户使用控制器（鼠标 / 触控屏）拖动（按压 / 触摸）时触发；而move事件会在控制器hover（onPointerMove）时即触发。而hover事件只处理控制器进入和离开事件（onPointerEnter & onPointerLeave）
>
> - useWheel和useScroll
>
>   wheel事件只会被鼠标触发。scroll事件只有在元素真实发生滚动的时候才会触发；而鼠标在元素上滑动滚轮就可以触发wheel事件。

**手势配置**

比较常用的配置项有：domTarget / eventOptions，以及用于控制手势范围的 bounds / distanceBounds / angleBounds / rubberband，用于控制swipe的 swipeDistance / swipeVelocity / swipeDuration 等。详细内容可以在[这里](t.music.163.com/st-playlist-summerize/summerize/index.html)找到。

> 其中：
>
> - domTarget用于替代bind函数，直接给dom ref绑定事件。当需要设置事件为[passive](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#improving_scrolling_performance_with_passive_listeners)来提升性能时，必须使用domTarget。

**事件**

recognizer会给事件回调传入丰富的事件信息，诸如：movement，offset，velocity等。详见[文档](https://use-gesture.netlify.app/docs/state/)。

> 其中较为常用的几个属性为：
>
> - movement和offset的区别
>
>   movement是单次拖动过程中手势拖动的向量，offset是在一个节点上所有手势拖动的向量和。
>
> - memo
>
>   用于记忆用户的自定义事件属性
>
> - cancel
>
>   取消当前事件
>
> - event
>
>   原始事件

**注意**

1. 为了避免和浏览器默认的拖动产生冲突（如：图片和链接的默认拖动行为），需要设置css：`touch-action: none`。同时，如需兼容firefox，还需要prevent default，详见[这里](https://use-gesture.netlify.app/docs/faq/#why-cant-i-properly-drag-an-image-or-a-link)。

### 实现

**初始化**

- 用户调用接口，传入handlers和config
- 实例化controller
  - 实例化接口对应的recognizer，将handlers传入recognizer
  - 将recognizer需要注册的事件列表注册到dom

**回调**

- 当recognizer监听到对应的手势时，触发handlers

### 性能优化

- Event delegation

  在react-use-gesture提供了更加丰富的事件信息的同时，也增加了每次事件计算信息的性能消耗。通过事件代理对事件进行统一处理可以优化性能。

## react-spring

### 简介

react spring提供了基于弹簧物理模拟的动画。它基于弹簧模型而不是常见的曲线 / 时长模型（尽管API中也支持指定动画时长），使其的动画效果更为自然。因为基于物理模型的动画将是**连贯**且更具**交互性**的。

> Andy Matuschak (ex Apple UI-Kit developer) [expressed it once](https://twitter.com/andy_matuschak/status/566736015188963328): *Animation APIs parameterized by duration and curve are fundamentally opposed to continuous, fluid interactivity*.

### 使用

**例子**

React-spring的使用主要分为三步：

1. 使用动画hook配置动画并获得返回的style属性和set函数
2. 将style属性传入animated组件
3. 使用set函数更新动画

```js
import {useSpring, animated} from 'react-spring'

function App() {
  const [style, set] = useSpring(() => ({opacity: 1, from: {opacity: 0}}))
  // ... use set to update style
  return <animated.div style={style}>I will fade in</animated.div>
}
```

> 注意：
>
> 1. set函数并不是直接设置style，而是提供了一个目标，而useSpring会自己计算出下一帧该如何变化以接近目标。
> 2. App函数并不会在set之后重新执行，因为style实际上是mutable的。
> 3. [另外一种更新动画的方式](https://www.react-spring.io/docs/hooks/use-spring#either-overwrite-values-to-change-the-animation:~:text=Either%3A%20overwrite%20values%20to%20change%20the%20animation)是在useSpring中直接传入新的值，但例子中使用的方法性能更优。

**动画hook**

| hook          | 描述                                                       |
| ------------- | ---------------------------------------------------------- |
| useSpring     | 动画化传入的参数                                           |
| useSprings    | 创建多个动画，使用各自的动画配置，用于静态列表             |
| useTrail      | 创建多个动画，使用同一个配置，每一个动画将会跟随前一个动画 |
| useTransition | 组件出现和消失的动画                                       |
| useChain      | 串联动画，下一个动画会在上一个结束后开始                   |

> 除了hook之外，react-spring还提供了动画组件，如：Spring，Trail等。参数与hook相似，这里不再赘述。

**动画属性**

- 字段类型（number + string）

  reat-spring不仅仅支持例子中的数字类型的动画，也支持字符串类型（如：transform，color）的动画。详见[Up-front interpolation](https://www.react-spring.io/docs/hooks/basics#up-front-interpolation:~:text=Up%2Dfront%20interpolation)。

- 插值（interpolate）

  同时，react-spring也支持使用[插值函数](https://www.react-spring.io/docs/hooks/api#interpolations:~:text=Interpolations)将一个更新后的数值映射到真实的动画属性。它允许用户**重复使用**一个计算结果，将其应用到多个动画属性上，提高了**性能**。一个例子可以在[这里](https://www.react-spring.io/docs/hooks/basics#view-interpolation:~:text=View%20interpolation,-In)找到。

**更新动画**

使用hook返回的set函数更新目标，参数同hook参数一致。见[properties](https://www.react-spring.io/docs/hooks/api#properties:~:text=Properties,100%2C%20onRest%3A%20()%20%3D%3E%20...%20%7D)。

- from / to ...

- [config](https://www.react-spring.io/docs/hooks/api#configs:~:text=Configs)

  通过 mass / tension / friction 设置基于物理模拟的动画，或使用duration来设置基于时长的动画。

  或者可以使用config.default / config.slow ... 等预设动画。

### 实现

- style：通过将css属性与动画组件直接绑定，跳过了业务组件的重新渲染，提高性能。

  **监听动画属性**：React-spring的hooks返回的style属性并不是简单的`CSSProperties`类型的数据，而是`SpringValues`类型。其中每个`SpringValue`通过[fluids](https://www.npmjs.com/package/fluids)库的`addFluidObserver`函数监测变化。实现类似mobx中observable的效果。当style被传入animated组件时，`getAnimatedState`函数会提取props中的fluidValue并添加到依赖集中，并为每个依赖通过`addFluidObserver`监听变化。

  **更新**：当`SpringValue`更新时，会被加入一个`Set`。在每个动画帧（使用`rafz`库的`onFrame`函数，调用requestAnimationFrame）的最后，这些更新会被读取并通过`flushCalls`函数应用到组件上。

## 兼容性问题

### Pointer event

**问题**

由于react-use-spring使用了[pointer event](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent)，导致在低版本的浏览器中将无法使用。

**解决方案**

所幸[pepjs](https://github.com/jquery/PEP)提供了pointer event的polyfill方案，而其使用方式也十分简单：

1. 安装pepjs

   ```bash
   npm install pepjs
   ```

2. 在代码中引入

   ```javascript
   import 'pepjs'
   ```

3. 在对应的节点设置touch-action属性（浏览器为了优化性能，不指定touch-action默认不触发事件）

   ```html
   <button id='test' touch-action="none">Test button!</button>
   ```

4. 注册事件

   - react

     ```react
     export function Pointable() {
       return <div touch-action="none" onPointerDown={(e) => console.log(e)} /> 
     }
     ```

   - DOM

     ```
     document.getElementById("test").addEventListener("pointerdown", function(e) {console.log(e)}
     ```

注意：在使用pepjs时，当用户的触摸操作需要用到浏览器默认的行为（如：scroll）时，同样需要指定touch-action（如：touch-action='auto'）。
### IOS 13.0 & 13.1

**问题**

在IOS13.x及之后的版本中，移动端的safari&chrome开始支持pointer event。但是对于13.0 & 13.1，官方对于pointer event的实现（pointerevent.buttons的值）仍然有问题，见[can i use](https://caniuse.com/pointer)。在这两个版本中，由于：

1. pepjs不再polyfill
2. react-use-gesture的实现依赖于pointerevent.buttons的值

造成了在13.0 & 13.1产生了一个pointerevent.buttons polyfill的真空地带，导致react-use-gesture失效。这个问题在[issue](https://github.com/pmndrs/react-use-gesture/issues/103#issuecomment-545422374)中也有描述。

**解决方案**

因为这是pointer event实现的bug，react-use-gesture并不准备通过为这两个版本修改代码。因此我们需要自己fork一个react-use-gesture:

路径：`./src/recognizers/DragRecognizer.ts`

```diff
onDragChange = (event: PointerEvent): void => {

    ...

    // If the event doesn't have any button / touches left we should cancel
    // the gesture. This may happen if the drag release happens outside the browser
    // window.
-   if (!genericEventData.down) {
+   if (!is_ios_13_0_or_13_1() && !genericEventData.down) {
      this.onDragEnd(event)
      return
    }

    ...
}
```

**注意**

修改后的库会有一个问题，即：使用鼠标拖动移出浏览器窗口后释放鼠标，再移入时，在IOS13.0和IOS13.1中，将继续处于拖动状态。

## 踩坑

### react-spring v9 资源汇总

React-spring目前（2021/02/07）最新的稳定版本是8.0.27，最新的release candidate是9.0.0-rc.3。但是由于官网的文档仍然是v8的版本，且我们在使用v9时遇到了很多奇怪的问题，并不建议使用v9。

如果你仍然想要使用v9，以下是一些文档的链接：

- [v9新特性](https://aleclarson.github.io/react-spring/v9/)
- [v9 useTransition的新API](https://aleclarson.github.io/react-spring/v9/breaking-changes/)

### react-spring v9 常见问题及解决方式

- Q：在开发环境无问题，部署环境出现TypeError（如： `Uncaught TypeError: r.willAdvance is not a function`）

  A：因为v9 - rc.3在package.json中声明了`sideEffect: ture`。webpack时会进行剪枝（tree shaking）操作，导致报错（官方表示会在rc.4修复）。[修复方法](https://github.com/pmndrs/react-spring/issues/1078#issuecomment-753032426)：

  ```json
  "scripts": {
      "react-spring-issue-1078": "find node_modules -path \\*@react-spring/\\*/package.json -exec sed -i.bak 's/\"sideEffects\": false/\"sideEffects\": true/g' {} +",
      "postinstall": "npm run react-spring-issue-1078",
  }
  ```

- Q：useTransition / Transition 中各种不符合预期的表现

  A：首先检查是否使用了[v9新的API格式](https://aleclarson.github.io/react-spring/v9/breaking-changes/#The-useTransition-hook)。然后尝试在items参数中传入数组而不是其他类型。尽管在[文档](https://www.react-spring.io/docs/hooks/use-transition)的示例中有很多非数组的例子，但这些都不是标准的使用方法，在v9中可能导致各种预期之外的动画效果。

- Q：useTransition / Transition 中的 Multi-stage transitions 的stage数量和设定不符

  A：检查是否两个阶段设定的状态没有变化，如：

  ```jsx
  leave={[
  	{opacity: 1},
    {opacity: 1},
    {opacity: 0},
  ]}
  ```

  上图的设定会导致中间的stage被无视，实际效果不是`opacity: 1->1->0`，而是`opacity: 1->0`。一个备用方案是设定用户无法感知的变化，如：

  ```jsx
  leave={[
    {opacity: 1},
    {opacity: 0.99},
    {opacity: 0},
  ]}
  ```
