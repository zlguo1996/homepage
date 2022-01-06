---
title: 【学习笔记】Service Worker
tags:
- code
- frontend
- service worker
---

## 什么是 Service Worker

Service Worker 是浏览器在后台独立于网页运行的脚本，它打开了通向不需要网页或用户交互的功能的大门。 现在，它们已包括如[推送通知](https://developers.google.com/web/updates/2015/03/push-notifications-on-the-open-web)和[后台同步](https://developers.google.com/web/updates/2015/12/background-sync)等功能。 将来，Service Worker 将会支持如定期同步或地理围栏等其他功能。

### 注意事项

- 它是一种 [JavaScript Worker](https://www.html5rocks.com/en/tutorials/workers/basics/)，无法直接访问 DOM。 Service Worker 通过响应 [postMessage](https://html.spec.whatwg.org/multipage/workers.html#dom-worker-postmessage) 接口发送的消息来与其控制的页面通信，页面可在必要时对 DOM 执行操作。
- Service Worker 是一种可编程网络代理，让您能够控制页面所发送网络请求的处理方式。
- Service Worker 在不用时会被中止，并在下次有需要时重启，因此，您不能依赖 Service Worker `onfetch` 和 `onmessage` 处理程序中的全局状态。 如果存在您需要持续保存并在重启后加以重用的信息，Service Worker 可以访问 [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)。

### 生命周期

![生命周期](https://developers.google.com/web/fundamentals/primers/service-workers/images/sw-lifecycle.png)

### 作用域

Service worker文件的位置决定了它响应的fetch事件的范围。例：

- 注册于`/sw.js`：作用域将是整个来源。 换句话说，Service Worker 将接收此网域上所有事项的 `fetch` 事件。
- 注册于`/example/sw.js`：则Service Worker 将只能看到网址以 `/example/` 开头（即 `/example/page1/`、`/example/page2/`）的页面的 `fetch` 事件。

### 更新

1. 更新您的服务工作线程 JavaScript 文件。 用户导航至您的站点时，浏览器会尝试在后台重新下载定义 Service Worker 的脚本文件。 如果 Service Worker 文件与其当前所用文件存在字节差异，则将其视为*新 Service Worker*。
2. 新 Service Worker 将会启动，且将会触发 `install` 事件。
3. 此时，旧 Service Worker 仍控制着当前页面，因此新 Service Worker 将进入 `waiting` 状态。
4. 当网站上当前打开的页面关闭时，旧 Service Worker 将会被终止，新 Service Worker 将会取得控制权。
5. 新 Service Worker 取得控制权后，将会触发其 `activate` 事件。

即，存在一个时刻，同时存在着 新 & 旧 Service Worker

## 开发

### 调试

`chrome://inspect/#service-workers`

### 注册

在页面中进行注册

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
      // Registration was successful
      console.log('ServiceWorker registration successful with scope: ', registration.scope);
    }, function(err) {
      // registration failed :(
      console.log('ServiceWorker registration failed: ', err);
    });
  });
}
```

### 生命周期

下面生命周期的使用介绍将以一个缓存管理应用场景的代码为例：

#### install

```js
var CACHE_NAME = 'my-site-cache-v1';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  // Perform install steps
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```

#### fetch / message

响应请求

```js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }
        return fetch(event.request);
      }
    )
  );
});
```

#### activate

常见任务是缓存管理。 您希望在 `activate` 回调中执行此任务的原因在于，如果您在安装步骤中清除了任何旧缓存，则继续控制所有当前页面的任何旧 Service Worker 将突然无法从缓存中提供文件。

```js
self.addEventListener('activate', function(event) {

  var cacheAllowlist = ['pages-cache-v1', 'blog-posts-cache-v1'];

  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheAllowlist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

## 应用

### 请求缓存

#### 缓存策略

**Stale-While-Revalidate**

优先返回缓存内容，同时会触发一次请求

![](https://developers.google.com/web/tools/workbox/images/modules/workbox-strategies/stale-while-revalidate.png)

**Cache First (Cache Falling Back to Network)**

优先返回缓存内容，无缓存时才会请求

![](https://developers.google.com/web/tools/workbox/images/modules/workbox-strategies/cache-first.png)

**Network First (Network Falling Back to Cache)**

优先发送请求，请求失败时才会使用缓存

![](https://developers.google.com/web/tools/workbox/images/modules/workbox-strategies/network-first.png)

**Network Only**

只使用网络请求结果

![](https://developers.google.com/web/tools/workbox/images/modules/workbox-strategies/network-only.png)

**Cache Only**

只使用缓存结果

![](https://developers.google.com/web/tools/workbox/images/modules/workbox-strategies/cache-only.png)

## 相关概念

### HTTP Cache

#### HTTP Headers

- [`Cache-Control`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control#Browser_compatibility)：服务器可以返回Cache-Control来制定，在多长时间内使用何种缓存策略。这种策略指定了浏览器和其他中间缓存该如何缓存单个请求。
- [`ETag`](https://developer.mozilla.org/docs/Web/HTTP/Headers/ETag#Browser_compatibility)：基于内容的过期判断 - 当浏览器发现了一个过期的缓存，它会发送一个小token（通常是文件内容的哈希）来判断文件是否发生变化。如果服务器返回了同样的token，则说明文件内容没有发生变化，无需重新下载。
- [`Last-Modified`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Last-Modified#Browser_compatibility)：基于时间的过期判断 - 和ETag的目的相同，只不过基于时间来判断资源是否变化。

#### 流程图

![](https://web-dev.imgix.net/image/admin/htXr84PI8YR0lhgLPiqZ.png?auto=format&w=650)

## References

- [Service Worker: 简介](https://developers.google.com/web/fundamentals/primers/service-workers)
- [Workbox Strategies](https://developers.google.com/web/tools/workbox/modules/workbox-strategies#stale-while-revalidate)
- [后台同步 Introducing Background Sync](https://developers.google.com/web/updates/2015/12/background-sync)
- [推送通知 Web Push Notifications: Timely, Relevant, and Precise](https://developers.google.com/web/updates/2015/03/push-notifications-on-the-open-web)

- [Prevent unnecessary network requests with the HTTP Cache](https://web.dev/http-cache/)
