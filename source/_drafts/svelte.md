---
title: 【学习笔记】Svelte
tags:
- code
- frontend
- svelte
- framework
- study note
---

## 优点

- Write less code：代码量小，这得益于svelte设计的几个特性

  - Top-level elements

    react和vue只能有一个顶层元素（即便使用react的fragment也会造成额外的一层缩进）。而svelte的每个组件可以定义任意数量的顶层元素。

  - Bindings

    在react，当我们需要响应用户输入的时候，需要通过setState的函数对数据进行更新。数据和input的双向绑定不是那么直接。而svelte使用和vue v-model属性类似的方法可以更加方便地实现双向绑定。

  - State

    react通过useState hook对状态进行更新。而svelte只要通过赋值就可以更新状态。

- No virtual DOM

  virtual DOM的diff函数会耗费一些性能，同时经常会重复计算一些没必要再计算的数据（如果要避免重复计算，又会需要使用memo这些方式来增加代码量和复杂度）。svelte将代码编译成无需引入框架代码的纯JS，让app能更快地启动，以及保持快速地运行速度。

- Truely reactive

  无需复杂的状态管理库，svelte会编译出响应式的JS代码。利用svelte预编译的优势，无需再使用setState这些函数来通知框架状态被更新，可以直接使用赋值操作来进行更新。

## 潜在问题

1. 代码量小并不绝对：虽然在简单的 demo 里面代码量确实非常小，但同样的组件模板，这样的 imperative 操作生成的代码量会比 vdom 渲染函数要大，多个组件中会有很多重复的代码（虽然 gzip 时候可以缓解，但 parse 和 evaluate 是免不了的）。项目里的组件越多，代码量的差异就会逐渐缩小。同时，并不是真正的如宣传的那样 “没有 runtime“，而是根据你的代码按需 import 而已。使用的功能越多，Svelte 要包含的运行时代码也越多，最终在实际生产项目中能有多少尺寸优势，其实很难说。

2. 性能优势不明显：Svelte 在大型应用中的性能还有待观察，尤其是在大量动态内容和嵌套组件的情况下。它的更新策略决定了它也需要类似 React 的 shouldComponentUpdate 的机制来防止过度更新。另一方面，其性能优势比起现在的主流框架并不是质的区别，现在大部分主流框架的性能都可以做到 [vanilla js](https://www.zhihu.com/search?q=vanilla+js&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A133912199}) 的 1.2~1.5 倍慢，基于 Virtual DOM 的 Inferno 更是接近原生，证明了 Virtual DOM 这个方向理论上的可能性，所以可以预见以后 web 的性能瓶颈更多是 DOM 本身而不是框架。

3. 无法享用vdom的好处：Svelte 的编译策略决定了它跟 Virtual DOM 绝缘（渲染函数由于其动态性，无法像模板那样可以被可靠地静态分析），也就享受不到 Virtual DOM 带来的诸多好处，比如基于 [render function](https://www.zhihu.com/search?q=render+function&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A133912199}) 的组件的强大抽象能力，基于 vdom 做测试，服务端/原生渲染亲和性。

> 引自[回答](https://www.zhihu.com/question/53150351/answer/133912199)

## 运行

### 创建工程

```bash
npx degit sveltejs/template my-svelte-project
cd my-svelte-project
npm install
npm run dev
```

### 组件格式

**文件**

Sevlte 组件被写入以`.svelte` 为后缀的文件中。svelte的语法是HTML的超集。它包含可选的三个部分 - 脚本（script）、样式（style）、标记（markup）。

```html
<script>
	// logic goes here
</script>

<!-- markup (zero or more items) goes here -->

<style>
	/* styles go here */
</style>
```

#### \<script>

script标签包含了组件实例被创建时需要运行的JS代码。再最顶层声明或引入的变量再同组件的markup中可见。同时它包含四条额外的规则：

- `export` creates a component prop
- Assignments are 'reactive'
- `$:` marks a statement as reactive
- Prefix stores with `$` to access their values

#### \<script context="module">

标记了context=“module”的script标签只会在模块第一次执行的时候被执行（而不是每次创建一个实例的时候都被执行）。这个标签内部声明的变量可以在普通的script标签中被访问，但反之普通script中的变量不可以在本标签的脚本中被访问。

在这个标签中的export会成为该编译后模块的export（无法使用export default，因为default就是组件本身）。

#### \<style>

style标签内的CSS代码作用域会被限定在该组件内。（局部作用域是通过给受影响的元素添加class来实现的，class名通过组件style的哈希生成，如：svelte-123xyz）

通过:global(...)可以让style作用于全局域。通过-global-前缀可以让@keyframes在全局被访问。

每个组件最多只可有一个顶级style标签。但元素内部可以嵌套style标签，它将会被直接插入DOM，此时不会应用局部作用域。

## 原理

svelte 的源码由两大部分组成，compiler 和 runtime。compiler 的作用是将 svelte 模版语法编译为浏览器能够识别的js SvelteComponent，而 runtime 则是在浏览器中帮助业务代码运作的运行时函数。

### complier

Svelte 如其介绍所说，在 complier 阶段完成了大部分的工作，而 complier 又分为 parse 和 complie 两部分：

#### parse

parse 会读取 `.svelte` 文件的内容进行解析。

- 对于 html 部分的内容，会分为 tag、mustache、text 三种解析类型：

- - tag: tag 解析的内容以 `<` 作为标识，包括 HTMLElement、style标签、script标签以及用户自定义的 svelte 组件以及 svelte 实现的一些特殊标签如 `svelte:head` 、`svelte:options` 、`svelte:window` 以及`svelte:body` 等。
  - muscache：mustache 以 `{` 作为标识，识别的内容除了模板语法之外，还包括 svelte 的逻辑渲染(else……if、each)等语法、`{``@html``}`、`{``@debug}` 等。
  - text就是无语义的静态文本

最终parse会将`.svelte` 的内容解析成含有 `html` 、`css` 、`instance` 、`module` 四部分的ast。

Instance 是指 script 标签中响应式的属性和方法，module 是使用 `<script context="module"` 声明 的无响应的变量和方法。

#### complie

Complie 首先会将 parse 过程中拿到的语法树（包含 html，css，instance 和 module）转换为 Component，然后在 render_dom 中通过 code-red 中的 print 函数将component 的转换为 js 可运行代码，最终输出 complier 的结果。

### runtime

生成的运行时主要包含：（例子可见Code-Runtime部分）

1. 生成的`create_fragement` 和 `instance` 两个方法
2. 从 `svelte/internal` 引入 `append`、`detach` 、`element` 、`insert` 、`listen` 等方法，这些方法都是对原生 dom 操作的封装

#### create_fragment

`create_fragment` 是和每个组件生成 dom 相关的方法，里面定义了 `c` 、 `m` 、`p` 、`i` 、`o` 、`d` 等一系列内置方法。

- c(create)：create 函数里会对一系列的 dom 节点进行创建，并将它们保存在 create_fragment 的闭包中。
- m(mount)：mount 函数中会对根据 dom 节点的层级关系，构建 dom 树，并将 dom 树插入到页面 target 节点下。同时还会将 dom 上相关的事件监听进行绑定，然后将组件的 mounted 状态置为 true。
- p(update)：组件的状态发生改变时会触发 update 函数，对 dom 中相应的数据重新进行更新渲染。
- d(distroy)：将 dom 挂载从页面中移除，将组件的 mounted 状态置为 false，同时移除一系列的事件监听。

#### instance

`instance` 方法中返回了包含组件实例中属性和方法的数组，将相应的数据绑定在组件实例的 `$$.ctx` 上，并且根据用户定义的触发属性修改的方法去调用一个 `$$invalidate`方法。

在invalidate中，使用了一个 `make_dirty` 方法。svelte 是通过如下操作对属性进行脏标记的：在dirty 数组中存储一系列的 32 位整数，通过这一操作，提高了内存利用率，每个数组项可以存储31个属性是否需要更新。

> 例如如下32位的整数43，对应的32位二进制为：
>
> Dirty = [43]
>
> 43 -> 0000 0000 0000 0000 0000 0000 0010 1011
>
> 二进制中为1的位代表需要更新的 instance 中数组第几项，即第1、2、4、6项属性需要更新。

## References

- [官网](https://svelte.dev/)
- [教程](https://svelte.dev/tutorial/basics)
- [文档](https://svelte.dev/docs)

- 如何看待 svelte 这个前端框架？ - 尤雨溪的回答 - 知乎 https://www.zhihu.com/question/53150351/answer/133912199
- [Svelte 原理浅析与评测](https://mp.weixin.qq.com/s/YuH13pD1Dp98HfPCIjer7w)

## Code

### Runtime

编译前：

```html
<script>

  let count = 0;

  const addCount = () => {

    count += 1;

  };

</script>



<div>

  <button on:click={addCount}>增加</button>

  <p>count is: {count}</p>

</div>
```

编译后：

````js
/* App.svelte generated by Svelte v3.42.4 */

import {

  SvelteComponent,

  append,

  detach,

  element,

  init,

  insert,

  listen,

  noop,

  safe_not_equal,

  set_data,

  space,

  text,

} from 'svelte/internal';



function create_fragment(ctx) {

  let div;

  let button;

  let t1;

  let p;

  let t2;

  let t3;

  let mounted;

  let dispose;



  return {

    c() {

      div = element('div');

      button = element('button');

      button.textContent = '增加';

      t1 = space();

      p = element('p');

      t2 = text('count is: ');

      t3 = text(/*count*/ ctx[0]);

    },

    m(target, anchor) {

      insert(target, div, anchor);

      append(div, button);

      append(div, t1);

      append(div, p);

      append(p, t2);

      append(p, t3);



      if (!mounted) {

        dispose = listen(button, 'click', /*addCount*/ ctx[1]);

        mounted = true;

      }

    },

    p(ctx, [dirty]) {

      if (dirty & /*count*/ 1) set_data(t3, /*count*/ ctx[0]);

    },

    i: noop,

    o: noop,

    d(detaching) {

      if (detaching) detach(div);

      mounted = false;

      dispose();

    },

  };

}



function instance($$self, $$props, $$invalidate) {

  let count = 0;



  const addCount = () => {

    $$invalidate(0, (count += 1));

  };



  return [count, addCount];

}



class App extends SvelteComponent {

  constructor(options) {

    super();

    init(this, options, instance, create_fragment, safe_not_equal, {});

  }

}



export default App;
````

