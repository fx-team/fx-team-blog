---
title: Service Worker 初体验
date: 2018-01-28 23:33:03
tags: Service Worker sw caches
---
## 一、什么是 PWA？
Progressive Web App, 简称 PWA，是提升 Web App 的体验的一种新方法，能给用户原生应用的体验。

PWA 能做到原生应用的体验不是靠特指某一项技术，而是经过应用一些新技术进行改进，在安全、性能和体验三个方面都有很大提升，PWA 本质上是 Web App，借助一些新技术也具备了 Native App 的一些特性，兼具 Web App 和 Native App 的优点。

**关键技术之一：Service Worker**

当用户打开我们站点时（从桌面 icon 或者从浏览器），通过 Service Worker 能够让用户在网络条件很差的情况下也能瞬间加载并且展现。

Service Worker 是用 JavaScript 编写的 JS 文件，能够代理请求，并且能够操作浏览器缓存，通过将缓存的内容直接返回，让请求能够瞬间完成。开发者可以预存储关键文件，可以淘汰过期的文件等等，给用户提供可靠的体验。
<!-- more -->
## 二、技术铺垫
#### 1、全局作用域self
``` js
window === self                  // true
window.window === window.self    // true
window.self === self             // true
window.window === self           // true
```
从上图可以看出传统 web 页面中，window 和 self是完全相等的。

传统的web页面的JavaScript脚本是单线程的，这个线程主要与浏览器窗口打交道，主要作用就是实现浏览器窗体内的元素交互效果，因此只要是全局对象，都可以使用 window 对象来获取。

Service Worker 是运行在浏览器上开辟的一个新线程，浏览器背后悄悄运行的线程，所以没有 window 对象，会使用 self 获取当前运行环境的上下文，即使用 self 来表示全局作用域。

#### 2、Cache Storage
Cache Storage 可以用变量 caches 来引用，以下是 Cache Storage 的主要 API介绍
* caches.open(cacheName) 用于获取一个 Cache 对象实例。
* caches.match() 用于检查 CacheStorage 中是否存在以Request 为 key 的 Cache 对象。
* caches.has() 用于检查是否存在指定名称的 Cache 对象。
* caches.keys() 用于返回 CacheStorage 中所有 Cache 对象的 cacheName 列表。
* caches.delete() 用于删除指定 cacheName 的 Cache 对象。
Cache Storage 通过 cacheName 标记缓存版本，所以就会存在多个版本的 Cache Storage 资源。为什么需要 cacheName 来标记版本呢？

假设当前域名下所有的覆盖式发布的静态资源和接口数据全部存储在同一个 cacheName 里面，业务部署更新后，无法识别旧的冗余资源，单靠前端无法完全清除。这是因为 Service Worker 不知道完整的静态资源路径表，只能在客户端发起请求时去做判断，那些当前不会用到的资源不代表以后一定不会使用到。假如静态资源是非覆盖式发布，那么冗余的资源就更多了。这里要特别注意的是，Cache 不会过期，只能显式删除。

#### 3、Cache
规范里 Cache 对应内核的 ServiceWorkerCache 对象，提供了已缓存的 Request / Response 对象体的存储管理机制。它提供了一系列管理存储的 JS 接口：
* cache.put() 用于把 Request / Response 对象体放进指定的 Cache。
* cache.add() 用于获取一个 Request 的 Response，并将 Request / Response 对象体放进指定的 Cache。注：等价于 fetch(request) + Cache.put(request, response)。
* cache.addAll() 用于获取一组 Request 的 Response，并将该组 Request / Response 对象体放进指定的Cache。
* cache.keys() 用于获取 Cache 中所有 key。
* cache.match() 用于查找是否存在以 Request 为 key 的 Cache 对象。
* cache.matchAll() 用于查找是否存在一组以 Request 为 key 的 Cache 对象组。
* cache.delete() 用于删除以 Request 为 key 的 Cache Entry。注意，Cache 不会过期，只能显式删除 。

## 三、Service Worker
#### 1、Service Worker概念和用法
我们平常浏览器窗口中跑的页面运行的是主JavaScript线程，DOM和window全局变量都是可以访问的。而Service Worker是走的另外的线程，可理解为在浏览器背后默默运行的一个线程，脱离浏览器窗体，因此，window以及DOM都是不能访问的，此时我们可以使用之前讲到的self访问全局上下文。

由于Service Worker走的是另外的线程，因此，Service Worker不会阻塞主JavaScript线程，也就是不会引起浏览器页面加载的卡顿之类。同时，由于Service Worker设计基于Promise，完全异步，同步API（如XHR和localStorage）不能在Service Worker中使用。

Service Worker对我们的协议也有要求，就是必须是https协议的，不过本地开发Service Worker在http://localhost或者http://127.0.0.1这种本地环境可以直接运行。如果想线上真是环境预览，可以考虑借助Github pages，因为它是https协议的。

#### 2、Service Worker生命周期
Service Worker注册时候的生命周期是这样的：
1.	Download – 下载注册的JS文件
2.	Install – 安装
3.  Activate – 激活

一旦安装完成，如果注册的JS没有变化，则直接显示当前激活态。

用线把整个流程链接起来就是下面这样： 

注册成功-installing--------------> 安装成功（installed）(waiting) ---------activating----------> 激活成功 (activated)------> 销毁（redundant）

其中任何一个步骤失败都将进入销毁（redundant）

#### 2、Service Worker 对应的事件名
1. self.addEventListener('install', function(event) { /* 安装后... */ });
2. self.addEventListener('activate', function(event) { /* 激活后... */ });
3. elf.addEventListener('fetch', function(event) { /* 请求后... */ });

install 事件是服务工作线程获取的第一个事件，并且它仅发生一次。

传递到 installEvent.waitUntil() 的一个 promise 可表明安装的持续时间以及安装是否成功。

在成功完成安装并处于“activate活动状态”之前，服务工作线程不会收到 fetch 和 push 等事件。

默认情况下，不会通过服务工作线程获取页面，除非页面请求本身需要执行服务工作线程。 因此，您需要刷新页面以查看服务工作线程的影响。

clients.claim() 可替换此默认值，并控制未控制的页面。

## 四、简单的 PWA 方案
**主要使用的技术**
1. App Manifest 
2. Service Worker
3. cacheStorage

#### 1、 App Manifest
添加 manifest.json 文件。

为了让 PWA 应用被添加到主屏幕, 使用 manifest.json 定义应用的名称, 图标等等信息。

``` json
{
  "name": "pwa名称",
  "short_name": "pwa名称",
  "display": "standalone",
  "start_url": "/",
  "theme_color": "#8888ff",
  "background_color": "#aaaaff",
  "icons": [
    {
      "src": "e.png",
      "sizes": "256x256",
      "type": "image/png"
    }
  ]
}
```
然后引入到html的head中，
``` html
<link rel="manifest" href="manifest.json" />
```

#### 2、Service Worker
主要操作是：
1. 注册完成安装 Service Worker 时, 抓取资源写入缓存； 
2. 网页抓取资源的过程中, 在 Service Worker 可以捕获到 fetch 事件, 编写代码如何响应资源的请求；
3. 最后一步是更新静态资源的功能。
``` js
if (navigator.serviceWorker != null) {
	navigator.serviceWorker.register('sw.js')
	.then(function(registration) {
		console.log('Registered events at scope: ', registration.scope);
	});
}
```
``` js
// sw.js
var cacheStorageKey = 'minimal-pwa-1'

var cacheList = [
  '/',
  "index.html",
  "style.css",
  "img.png"
];

// 抓取资源写入缓存
self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(cacheStorageKey)
    .then(cache => cache.addAll(cacheList))
    .then(() => self.skipWaiting())
  )
});

// 捕获到 fetch 事件, 编写代码如何响应资源的请求
self.addEventListener('fetch', function(e) {
  e.respondWith(
    caches.match(e.request).then(function(response) {
      if (response != null) {
        return response
      }
      return fetch(e.request.url)
    })
  )
});

//更新静态资源
self.addEventListener('activate', function(e) {
  e.waitUntil(
    Promise.all(
      caches.keys().then(cacheNames => {
        return cacheNames.each(name => {
          if (name !== cacheStorageKey) {
            return caches.delete(name)
          }
        })
      })
    ).then(() => {
      return self.clients.claim()
    })
  )
});
```

以上的功能都准备好就可以简单的生成一个 PWA 的网站了。可以使用支持 PWA 的手机进行预览，根据提示增加到桌面。

## 参考文献：
1. [服务工作线程：简介](https://developers.google.cn/web/fundamentals/primers/service-workers/?hl=zh-cn)
2. [Web 技术文档 Web API 接口 ServiceWorker](https://developer.mozilla.org/zh-CN/docs/Web/API/ServiceWorker)
3. [借助Service Worker和cacheStorage缓存及离线开发](http://www.zhangxinxu.com/wordpress/2017/07/service-worker-cachestorage-offline-develop/)
4. [网站渐进式增强体验(PWA)改造：Service Worker 应用详解](https://lzw.me/a/pwa-service-worker.html)
5. [PWA 入门: 理解和创建 Service Worker 脚本](https://zhuanlan.zhihu.com/p/25524382)
