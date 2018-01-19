# Quick Start


## 创建属于你的git信息
1. 本地创建一个`config.js`
2. 编辑`config.js`，进行配置
```
module.exports = {
  username: '', // 你的github username
  email: '' // 你的github email
};
```
### 为什么会有它
`hexo-deployer-git` **deploy**时会使用全局的`git config`，如果全局`git config`配置的工作`git`账号将会非常麻烦，提交信息会是你的工作账号。所以我们在`hexo-deployer-git`基础上进行了优化，得到了`hexo-deployer-git-fx`，它会读取`config.js`中的配置，这样就不会有`git`信息错乱的困扰了。
同时为了方便团队协作，`config.js`只在本地使用，不提交到服务端。
## Create a new post

``` bash
$ hexo new "My New Post"
```

新建一篇文章。如果标题包含空格的话，请使用引号括起来。

More info: [Writing](https://hexo.io/docs/writing.html)

## Generate static files
``` bash
$ hexo generate
```

生成静态文件。
> -d, --deploy	文件生成后立即部署网站

> -w, --watch	监视文件变动

More info: [Generating](https://hexo.io/docs/generating.html)

## Run server

``` bash
$ hexo server
```

启动服务器。默认情况下，访问网址为： http://localhost:4000/。

More info: [Server](https://hexo.io/docs/server.html)

## Deploy to remote sites
``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)

## 更多命令和写作操作请看官方网站说明
[https://hexo.io/zh-cn/docs/commands.html](https://hexo.io/zh-cn/docs/commands.html)

## Markdown 编写规范
[Markdown 编写规范链接](https://github.com/fx-team/styleguide/blob/master/markdown.md)

## Tips
1. 发布新文章 & 修改文章内容
``` bash
$ hexo g -d
```

2. 删除文章
```
$ hexo clean
$ hexo d
```

3. 修改发布人的配置
将`config.js`中的内容修改为自己的`github`账号信息，方便点击作者可以跳到相应页面。