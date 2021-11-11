---
title: 接入微前端需要注意什么
categories:
  - code
  - frontend
  - micro frontends
date: 2021-11-11 20:18:19
tags:
---


### 背景

[微前端（Micro-Frontends）](https://micro-frontends.org/)是一种类似于微服务的架构，它将微服务的理念应用于 Web 端。它将 Web 应用由单一的单体应用**拆解**为**功能**的**组合**。每个功能可以隶属于不同的团队，使用不同的前端架构，部署在不同的地址。

微前端来源于前端业务间**开发流程接耦**的需求。即希望在从开发到上线的整个业务流程上，各个团队能够独立进行互不干扰：

- 开发团队组织结构的演变：

  ![Monolithic Frontends](https://micro-frontends.org/ressources/diagrams/organisational/monolith-frontback-microservices.png)

- 微前端团队的组织结构：

  ![End-To-End Teams with Micro Frontends](https://micro-frontends.org/ressources/diagrams/organisational/verticals-headline.png)



### 定义

微前端是这样一种架构风格：将前端应用分解成一些更小、更简单的能够独立开发、测试、部署的小块，而在用户看来仍然是内聚的单个产品

> Micro Frontend is a pattern to emerge for decomposing frontend monoliths into smaller, simpler chunks that can be developed, tested and deployed independently, while still appearing to customers as a single cohesive product.
>
> 摘自 https://front-hub.rdstation.com.br/docs/microfrontend

即:

1. 对用户而言：是一个完整的单个产品
2. 对开发者而言：是多个独立交付的前端应用

### 例子

如下是一个典型的微前端页面的结构。在一个公共的主应用中挂载了两个独立的子应用。三个应用代码分别存储于不同仓库，同时也部署于不同的地址。

![image](https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/11412748117/497e/2320/4b0a/93b0a666012108c98fe4bb8285774d6e.png)

由于子应用资源在运行时进行加载，满足了子应用间独立交付，互不干扰的需求。

### 广义的微前端

如果我们把微前端看作一个**容器应用**将各子应用结合起来。那么广义来说，我们可以按照集成方式将微前端分为两类：

1. 构建时集成：如webpack的Code Splitting、npm包等方式（缺陷：发布阶段的耦合，无法独立交付）
2. 运行时集成： 
   1. 客户端集成：iframe、Web Components和其他复合的JS集成（如使用qiankun等微前端框架）方式
   2. 服务端集成：如 SSR 拼装模板，React18支持的[Streaming HTML和Selective Hydration](https://github.com/reactwg/react-18/discussions/37)

接下来我们主要讨论的是**运行时**使用**JS集成**场景下的微前端，其他的集成方式选型可以参考[这篇文章](https://zhuanlan.zhihu.com/p/96464401#:~:text=%E5%BE%AE%E5%89%8D%E7%AB%AF%E6%9E%B6%E6%9E%84%E4%B8%AD%E4%B8%80%E8%88%AC%E4%BC%9A%E6%9C%89%E4%B8%AA%E5%AE%B9%E5%99%A8%E5%BA%94%E7%94%A8%EF%BC%88container%20application%EF%BC%89%E5%B0%86%E5%90%84%E5%AD%90%E5%BA%94%E7%94%A8%E9%9B%86%E6%88%90%E8%B5%B7%E6%9D%A5)。

### 应用场景

得益于微前端**拆解**的特性，目前多用于复杂的web应用/web站点（如：中后台页面、阿里云页面、figma插件系统）。但由于其潜在的加载性能问题，多用于对网络资源不敏感的应用。

在云音乐中，内部应用（如：cms后台）、B端应用（如：创作者中心）都是比较适合落地微前端的场景。

在以下场景中我们可以应用这项技术：

1. 增量升级：允许渐进式重构。在历史系统中会存在一些过时的技术，而在新业务中使用这些技术会造成开发效率的降低。如：一些老的 cms 中使用 regular 框架，而现在云音乐的新业务都使用 react 进行开发。
2. 独立部署：缩小单应用功能范围，降低变更风险，提升部署效率。随着业务中需求的堆积，单体应用代码量增加，构建时间也会增加，造成每次部署的长时间的等待。如：平台 cms 中存在 200+个页面。
3. 团队自治：围绕业务功能纵向组建团队，而不是基于技术职能划分。一个业务中遇到的能力可能由不同团队来维护，使用同一个仓库会造成开发流程上的混乱；另一方面，同一个中台能力也可能提供给不同的业务方。如：用户中台 cms 提供了面向不同业务（心遇、云音乐）的用户管理能力。

### 微前端体系

![components](https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/11414823066/584d/ed01/1e73/ab6bcd7eaa07b41d4b3137062fd099c7.png)

[字节的这篇文章](https://mp.weixin.qq.com/s/D7vidJrA9v_Qx3zSFoNbOQ)提到了**微前端体系**：为了在企业级的业务中落地微前端，我们需要的不仅仅是一个**微前端框架**，而是一整套覆盖开发流程的完整体系。它包括：

#### 治理体系

管理平台 & 发布流程

- 应用管理：主子应用版本管理，入口地址
- 依赖管理：主子应用间的依赖关系

#### 开发配套

开发工具 & 流程

- 流程文档
- 集成联调：主应用独立调试、子应用独立调试、主子应用联合调试的方式

#### 运行时容器

微前端框架、iframe、web component...

- 应用加载

  - 入口文件格式：JS & HTML
  - 入口的注入方式：构建时注入 & 运行时注入
- 生命周期 - 加载 / 挂载 / 更新 / 卸载

  - 加载：请求资源
  - 挂载：初始化
  - 更新：路由变化、主子应用双向通信
  - 卸载：清理
- 沙箱隔离

  - JS隔离

    - snapshot：子应用挂载时对window进行快照，子应用卸载时恢复快照
    - wasm VM：子应用放在wasm的js解释器中执行（隔离过于严格，通信开销大）
    - [with() + new Function(code) + Proxy](https://mp.weixin.qq.com/s/D7vidJrA9v_Qx3zSFoNbOQ#:~:text=Function(code)%20%2B%20Proxy-,with%20%E8%AF%AD%E6%B3%95%E7%94%A8%E4%BA%8E%E6%94%B9%E5%8F%98%E4%BD%9C%E7%94%A8%E5%9F%9F%E9%93%BE,-%EF%BC%8C%E8%BF%99%E9%87%8C%E7%94%A8%E6%9D%A5)
      - with()：改变作用域，拦截对全局变量的查找。
      - new Function(code)：只能访问全局作用域（于此相对，eval可以访问局部变量）
      - Proxy：对document、history、location的操作做劫持

    - with() + new Function(code) + Proxy + iframe
      - 其他与上面一致，但是取iframe的window解决了上面对windows浅拷贝导致的全局API逃逸问题

  - CSS隔离

    - 切换应用时卸载（同一时刻应用间CSS还是会互相干扰）
    - shadow dom
      - 严格隔离了主子应用CSS。但是无法解决所有问题，如：弹窗无法应用子应用样式
      - hack：在document.body上的插入也应用shadow dom，并同步css。但是还有一些难以解决的问题：1. 两个shadow dom间样式的双向同步；2. css in js的动态插入；3. 插入dom其他位置难以劫持
- 路由同步

  - 主子应用共享浏览器历史，**主子应用**都**能够**操控和响应路由
    - history模式
      - 主响应子：劫持子应用history.pushState，主应用接收通知后replaceState。[参考](https://juejin.cn/post/6847902217945481224#:~:text=%E8%B7%AF%E7%94%B1%E9%80%BB%E8%BE%91%E3%80%82-,%E5%AD%90%E5%BA%94%E7%94%A8%E8%B7%AF%E7%94%B1%E5%90%8C%E6%AD%A5%E5%9B%9E%E4%B8%BB%E5%BA%94%E7%94%A8,-%E4%B8%BB%E5%BA%94%E7%94%A8%E6%A0%B9%E6%8D%AE)
      - 子响应主：劫持子应用popstate事件的监听，主应用路由变化后主动触发。[参考](https://juejin.cn/post/6991409477685624839#:~:text=DOM%20%E7%9A%84%E5%88%86%E6%9E%90%EF%BC%8C-,%E5%BE%AE%E5%BA%94%E7%94%A8%E6%98%AF%E9%80%9A%E8%BF%87%E4%B8%8B%E9%9D%A2%E4%B8%A4%E7%A7%8D%E6%96%B9%E5%BC%8F%E5%8C%B9%E9%85%8D%E5%AF%B9%E5%BA%94%E9%A1%B5%E9%9D%A2%E7%9A%84,-%E3%80%82)
    - hash模式
      - 响应hashchange事件
  - **子应用**能**正确**操控和响应路由（通过微前端访问和独立访问时子应用页面的url不同）
- 应用通信

  - 主子应用通信
- 异常处理

  - 加载失败，路由匹配失败...

#### 微物料

应用的拆分粒度 & 加载方式。具体的分类可见[这里](https://mp.weixin.qq.com/s/D7vidJrA9v_Qx3zSFoNbOQ#:~:text=%E5%9C%A8%E8%BF%99%E4%B8%AA%E5%9C%BA%E6%99%AF%E4%B8%8B%EF%BC%8C%E7%AE%80%E5%8D%95%E5%8C%BA%E5%88%86%E4%B8%8B%E7%9B%AE%E5%89%8D%E8%BF%99%E5%87%A0%E4%B8%AA%E7%A7%B0%E5%91%BC%E7%9A%84%E8%BE%B9%E7%95%8C)。

### 问题 / 难点

#### 载入速度

##### 流量负担（公共资源）

重复加载公共资源是微前端的隔离中附带的问题。微前端降低了耦合度，但提取公共资源以统一处理又会增加微应用间耦合度。

- [公共依赖](https://zhuanlan.zhihu.com/p/96464401#:~:text=%E4%BC%9A%E7%A0%B4%E5%9D%8F%E5%8D%8F%E4%BD%9C-,%E6%B5%81%E9%87%8F%E8%B4%9F%E6%8B%85,-%E7%8B%AC%E7%AB%8B%E6%9E%84%E5%BB%BA)：主子应用间、子应用间的公共资源，如：公共组件的JS、CSS，公共的错误、性能监控、埋点功能的代码。（解决方案：如使用html的external JavaScript，webpack5的module federation）

- 请求资源：在微前端中，主子应用经常会请求同一个接口，在首次加载时，实际上请求返回数据相同，需要避免重复请求带来的开销。（解决方案：如使用[SWR](https://swr.vercel.app/zh-CN)缓存请求结果，主应用使用props向子应用传递请求结果）

##### 加载链路

为了便于对依赖进行管理，通常我们会动态下发子应用配置而非在构建时注入，这也一定程度上延长了资源加载的链路，进而降低了页面加载速度。

#### 管理 、操作的复杂性

- 交付流程如何支持多应用（依赖管理，版本控制） 
- 质量保障（问题定位，独立发布的质量把控，开发规范）
