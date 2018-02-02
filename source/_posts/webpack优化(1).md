---
title: webpack优化(1)
date: 2018-02-02 16:30:58
tags: happypack
---

# webpack 优化

前一段时间一直在写后台管理系统（vue + elementUI + webpack），数下来，虽然不多，也有 3 个了；由于是单页应用，每次到发布的时候，都会耗费大量的时间对代码进行编译压缩，所以时不时都会思考着如何才能优化这个过程；谷歌和度娘就像是哆啦 A 梦的奇幻袋，总能给大家带来意想不到的惊喜，当然，这次也不例外。

## happypack

[npm](https://www.npmjs.com/package/happypack)的官方解释比较简单：通过并行转换文件以使 webpack 的构建速度更快；说白了就是利用多线程的优势。

它提供了一个插件和一个加载器，两个并用才能启用 happypack。

迫不及待了已经，赶紧上代码了 →_→

### 引入

`require('os')`模块想必大家都不陌生了，通过`os.cpus().length`，获得线程的长度，供 happypack 使用。

```js
const HappyPack = require('happypack');
const os = require('os');
// 启动线程池
const HappyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length });
```

### 插件+加载器

一个静态对象代表一个加载器入口，每个对象里面有重要的三个字段，`id`表示唯一标识，`threadPool`表示 happypack 开启的线程数量，`loaders`表示真正需要加载的加载器。

```js
module.exports = {
  plugins: [
    new HappyPack({
      id: 'jsx',
      threadPool: HappyThreadPool,
      loaders: ['babel-loader', path.join(__dirname, 'loader/debug-loader')]
    })
  ]
};
```

咦？？？大家是不是发现了什么 → 有点陌生的`debug-loader`加载器，话说起来，这个是涛哥提供的 idea。

引申一下，由于 mock 数据到后期异常庞大（夸张一下），被打包进生产环境的 mock 数据被涛哥无情的揪了出来，真挚的摆在我的面前，尘世间最痛苦的事情是我没有能力失去它（mock 数据），如果上天再给我一次机会的话，我会说：我终于知道用`debug-loader`了。

涛哥建议我参考公司内部的配置工具，在生产环境中，针对特定的引入参数，对引入文件进行剔除。

迫不及待了，上代码啊啊啊啊啊啊啊~

代码简简单单，`source`入参代表文件内容，`resourceQuery`是`import 'filepath?default'`中`?`后边的部分（包括`?`）。

我这里的判断逻辑是，带有`?debug`字样的引入文件，只有在非生产环境中的 mock 环境才被引入（有点拗口）。

```js
'use strict';

module.exports = function(source) {
  const _this = this;
  const { resourceQuery } = _this;
  if (resourceQuery === '?debug') {
    const { NODE_ENV, mock } = process.env;
    return NODE_ENV !== 'production' && !!mock ? source : '';
  }
  return source;
};
```

### 使用方法

重点在于，loaders 字段中的值`['happypack/loader?id=jsx']`，其中，通过`id=jsx`进行 happypack 插件中的加载器替换。

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        loaders: ['happypack/loader?id=jsx'],
        include: [
          resolve('src'),
          resolve('test'),
          resolve('node_modules/webpack-dev-server/client')
        ]
      }
    ]
  }
};
```

敬请期待下一节 webpack 优化(2) Dllplugin ^\_^
