---
title: koa源码解析(1)
date: 2018-02-01 15:55:09
tags: application.js
---

# koa

koa 是由 Express 原班人马打造的，致力于成为一个更小、更富有表现力、更健壮的 Web 框架。 使用 koa 编写 web 应用，通过组合不同的 generator，可以免除重复繁琐的回调函数嵌套， 并极大地提升错误处理的效率。koa 不在内核方法中绑定任何中间件， 它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。

安装、版本依赖请[点我点我](https://koa.bootcss.com/) ~ O(∩_∩)O 哈哈~

开篇怎能没有 hello world 呢？→_→

```js
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);

// 不满意 换个口味吧？
const http = require('http');
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

http.createServer(app.callback()).listen(3000);
```

## koa 图解

图中一层一层的洋葱圈，其实就是每一个中间件函数`middlewarefn(ctx, next)`，有的是系统中间件，有的是用户中间件，这取决于用户如何定义中间件在系统中的功能层级（都是通过 app.use()加载，koa 内部不封装任何中间件函数）。

路线分析：外层到内层-对 request 请求进行处理，内层再到外层-对 response 请求进行处理。

代码分析：

```js
// 某一个中间件函数
async middlewarefn (ctx, next) {
  // 外层到内层的代码
  ......
  await next();
  // 内层到外层的代码
  ......
}
```

{% asset_img koaStructure.png koa 结构 %}

## koa 核心文件

有且仅有 4 个（精简、流畅、易用）

* application.js
* context.js
* request.js
* response.js

### application.js

今天主要解剖一下这个货 ^\_^

继承自 Emitter 类，主要用于监听 error。

利用`require('debug')('koa:application')`模块，把所有的 debug 都输出在 koa:application 域下，便于查看。

构造函数的参数简单易懂：

* proxy false
* middleware []
* subdomainOffset 2
* env process.env.NODE_ENV || 'development'
* context 上下文对象
* request 请求级别对象
* response 请求级别对象

应用实例。从用户的角度出发，从事的工作是暴露给用户可使用的方法；

例如：

* listen()
* use()
* toJSON()
* inspect()

#### listen()

不言而喻，最基本的功能。可以看出，核心是调用了`this.callback()`方法。

```js
listen(...args) {
  debug('listen');
  const server = http.createServer(this.callback());
  return server.listen(...args);
}
```

#### use()

`require('is-generator-function')`模块，可以判定该函数是否是生成器函数，进而使用`require('koa-convert')`模块把生成器函数转换为可解析的 Promise 函数。有兴趣的同学可以研究一下这 2 个模块。

```js
if (isGeneratorFunction(fn)) {
  fn = convert(fn);
}
this.middleware.push(fn);
return this;
```

#### toJSON()

`require('only')`模块，返回该对象中指定的<key, value>，形成一个新的对象。参数可以使数组（数组中是 key），也可以是字符串（用空格隔开 key）。

```js
return only(this, ['subdomainOffset', 'proxy', 'env']);
```

#### inspect()

直观理解：返回 toJSON()方法。

```js
return this.toJSON();
```

从功能性角度来讲，他主要的使命是：

* callback()
* createContext()
* handleRequest()
* respond()
* onerror()

#### callback()

`require('koa-compose')`模块，把列表中的中间件按先后顺序用 Promise 封装；next()方法返回的恰恰是上一个函数执行的环境，只不过就是把中间件函数变成了 middleware 数组中的下一个，说白了就是递归执行。该模块的核心代码：

```js
 function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)
      }
    }
}
```

接着就是创建上下文环境 createContext()，然后执行 handlerequest()方法。

```js
callback()
  const fn = compose(this.middleware);
  if (!this.listeners('error').length) this.on('error', this.onerror);
  const handleRequest = (req, res) => {
    const ctx = this.createContext(req, res);
    return this.handleRequest(ctx, fn);
  };
  return handleRequest;
}
```

#### createContext()

乍一看，我的个神啊，这么多，密密麻麻，不用害怕，其实真正核心的就只有这几行：

把 application 的实例赋值给上下文 context，把 http 中的 req,res 赋值给上下文 context；接下来把 req，res 同时赋值给 request.js 和 response.js 暴露出来的静态对象（便于使用）。

```js
context.app = request.app = response.app = this;
context.req = request.req = response.req = req;
context.res = request.res = response.res = res;
request.ctx = response.ctx = context;
request.response = response;
response.request = request;
```

其余的代码很好理解，上下文对象 ctx 加入 cookie，ip, originalUrl, accept, state 参数。

```js
createContext(req, res) {
  const context = Object.create(this.context);
  const request = context.request = Object.create(this.request);
  const response = context.response = Object.create(this.response);
  context.app = request.app = response.app = this;
  context.req = request.req = response.req = req;
  context.res = request.res = response.res = res;
  request.ctx = response.ctx = context;
  request.response = response;
  response.request = request;
  context.originalUrl = request.originalUrl = req.url;
  context.cookies = new Cookies(req, res, {
  keys: this.keys,
  secure: request.secure
  });
  request.ip = request.ips[0] || req.socket.remoteAddress || '';
  context.accept = request.accept = accepts(req);
  context.state = {};
  return context;
}
```

#### respond()

分别根据 buffer、string 和 stream 流来区分对待；默认 JSON 封装数据，再顺便求一个字节长度。

```js
if (Buffer.isBuffer(body)) return res.end(body);
if ('string' == typeof body) return res.end(body);
if (body instanceof Stream) return body.pipe(res); // body: json
body = JSON.stringify(body);
if (!res.headersSent) {
  ctx.length = Buffer.byteLength(body);
}
res.end(body);
```

#### onerror()

只打印 stack，默认在开头加一个空格，自己也不能解惑。

```js
const msg = err.stack || err.toString();
console.error();
console.error(msg.replace(/^/gm, ' '));
console.error();
```

敬请期待下一节 koa 源码分析(2) request.js ^\_^
