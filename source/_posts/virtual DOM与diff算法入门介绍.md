---
title: virtual DOM 与 diff 算法入门介绍
date: 2018-04-02 15:14:20
tags:
author: <a href="https://github.com/xa2mc" target="_blank">xa2mc</a>
---
随着前端框架的流行，Vue 和 React 被越来越多的公司和团队使用，大家今天就跟着我一起来看看 virtual DOM 作为 Vue 和 React 的核心，它到底是什么，为什么会存在 virtual DOM，以及它是如何使用的，最后给大家简单介绍一些 diff 算法的实现。下面我们开始吧~
通过今天的介绍，我们将了解以下三部分的内容，也希望大家看后有所收获~

1. virtual DOM 是什么，为什么会存在 virtual DOM ？

2. virtual DOM 如何应用，有哪些核心的 API ？

3. 简单介绍一下 diff 算法。

#### virtual DOM 是什么，为什么会存在 virtual DOM？
* 用 JS 模拟 DOM 结构（不是真正的DOM）；
* DOM 结构的变化，放在 JS 层来实现；
* 提高重绘性能；

简单总结一下，由于在浏览器端频繁操作 DOM 是非常耗性能的事情，为了避免这种情况，我们会使用 JS 来模拟 DOM 结构，同时，DOM 结构的变化也同样放在 JS 层操作（JS 是图灵完备语言），这就是为什么会存在 virtual DOM 的原因。
既然 DOM 结构需要用 JS 来进行模拟，那我们下面就举一个具体的例子看看究竟是如何进行模拟的呢？

``` html
<ul id='list'>
  <li class='item'>item 1</li>
  <li class='item'>item 2</li>
</ul>
```
上述是一个简单的列表结构，如果要将它用进行 JS 模拟，便是下面的结果：

``` javascript
{
  tag: 'ul',
  attrs: {
    id: 'list'
  },
  children: [{
    tag: 'li',
    attrs: {className: 'item'},
    children: ['item 1']
  }, {
    tag: 'li',
    attrs: {className: 'item'},
    children: ['item 2']
  }]
}
```
很明显，下面的对象包含了上述列表结构的全部信息，标签、属性、子节点等等，这样我们就完成了 virtual DOM 的初始工作，那么 virtual DOM 到底是怎么实现的呢，不急，我们先去看看在没有它出现之前，用 jQuery 是怎样实现的~

##### DEMO
需求：将下面的数据展示成表格，随便修改一个表格，表格也跟着修改；

``` javascript
[
  {
    name: '张三',
    age: '20',
    address: '北京'
  },
  {
    name: '李四',
    age: '21',
    address: '上海'
  },
  {
    name: '王五',
    age: '22',
    address: '广州'
  }
]
```
实现如下：

``` html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
    <div id="container"></div>
    <button id="btn-change">change</button>

    <script type="text/javascript" src="https://cdn.bootcss.com/jquery/3.2.0/jquery.js"></script>
    <script type="text/javascript">
        // 数据源
        var data = [
            {
                name: '张三',
                age: '20',
                address: '北京'
            },
            {
                name: '李四',
                age: '21',
                address: '上海'
            },
            {
                name: '王五',
                age: '22',
                address: '广州'
            }
        ]
        // 渲染函数
        function render(data) {
            // 外层容器
            var $container = $('#container');
            // 清空容器
            $container.html('');
            // 拼接 table
            var $table = $('<table>');
            $table.append($('<tr><td>name</td><td>age</td><td>address</td>/tr>'));
            data.forEach(function (item) {
                $table.append($('<tr><td>' + item.name + '</td><td>' + item.age + '</td><td>' + item.address + '</td>/tr>'))
            });
            // 渲染到页面（注意顺序问题）
            $container.append($table);
        }

        // 改变数据
        $('#btn-change').click(function () {
            data[1].age = 30
            data[2].address = '深圳'
            // re-render 再次渲染
            render(data);
        });

        // 页面加载完立刻执行（初次渲染）
        render(data);
    </script>
</body>
</html>
```
上面的代码实现逻辑比较清晰，也并不复杂，简单来说就是初次渲染，如果数据有改动，那么更新数据再次渲染，相信大家看完代码也都足够清楚。
做了足够的铺垫，下面正式的主人公要登场了，我们接着去解决上面提出的第二个问题，virtual DOM 如何应用，有哪些核心的 API ？
#### virtual DOM 如何应用，有哪些核心的 API ？
下面给大家介绍 snabbdom（ https://github.com/snabbdom/snabbdom ）github 上面关于 virtual DOM 的第三方开源库并不是很多，这也侧面反应了其实它的实现过程还是相对复杂的，之所以挑选了 snabbdom，也是因为 Vue 2.0 的 virtual DOM 就是在它的基础上实现的。

{% asset_img snabbdom.jpg 百度 cookie 示例 %}
上面这张图是 github 上面 snabbdom 的官方示例，重复出现的函数我已经标红了，不难猜测，h 函数和 patch 函数也正是 snabbdom 的核心函数。
我们仿照官方示例，试图改写刚才的 DEMO，随后去浏览器中查看 DOM 元素的变化，看看是否与我们预期的相一致。

``` html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
    <div id="container"></div>
    <button id="btn-change">change</button>

    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/snabbdom.js"></script>
    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/snabbdom-class.js"></script>
    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/snabbdom-props.js"></script>
    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/snabbdom-style.js"></script>
    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/snabbdom-eventlisteners.js"></script>
    <script src="https://cdn.bootcss.com/snabbdom/0.7.0/h.js"></script>
    <script type="text/javascript">
        var snabbdom = window.snabbdom
        // 定义关键函数 patch
        var patch = snabbdom.init([
            snabbdom_class,
            snabbdom_props,
            snabbdom_style,
            snabbdom_eventlisteners
        ])

        // 定义关键函数 h
        var h = snabbdom.h

        // 原始数据
        var data = [
            {
                name: '张三',
                age: '20',
                address: '北京'
            },
            {
                name: '李四',
                age: '21',
                address: '上海'
            },
            {
                name: '王五',
                age: '22',
                address: '广州'
            }
        ]
        // 把表头也放在 data 中
        data.unshift({
            name: '姓名',
            age: '年龄',
            address: '地址'
        })

        var container = document.getElementById('container')

        // 渲染函数
        var vnode
        function render(data) {
            var newVnode = h('table', {}, data.map(function (item) {
                var tds = []
                var i
                for (i in item) {
                    if (item.hasOwnProperty(i)) {
                        tds.push(h('td', {}, item[i] + ''))
                    }
                }
                return h('tr', {}, tds)
            }))

            if (vnode) {
                // re-render
                patch(vnode, newVnode)
            } else {
                // 初次渲染
                patch(container, newVnode)
            }
            // 存储当前的 vnode 结果
            vnode = newVnode
        }
        // 初次渲染
        render(data)

        var btnChange = document.getElementById('btn-change')
        btnChange.addEventListener('click', function () {
            data[1].age = 30
            data[2].address = '深圳'
            // re-render
            render(data)
        })
    </script>
</body>
</html>
```
上面的实现过程基本与 jQuery 的相类似，只不过引用了 snabbdom 中的函数，大家可以去浏览器中观察 DOM 的变化，看看与之前的有什么不同，不同的地方也恰恰就是 virtual DOM 存在的原因。
讲了 virtual DOM  是什么，也初步体验了 virtual DOM 的实现过程，接下来，我们继续去了解 virtual DOM 中的核心算法 —— diff 算法。
#### 简单介绍一下 diff 算法；
想提前说一点注意，diff 算法我在这里去繁就简，因为 diff 算法非常之复杂，源码量也非常之大，所以我们在这里只做最核心流程的介绍，不去关心具体的细节，如果有感兴趣的同学可以自己抽时间去研究更加深入的 diff 算法的实现。

通过上面 snabbdom 的了解，我们不难看出，patch 函数实际就是在实现 diff 算法，那么我们就抓住核心，去研究一下 patch 函数是如何实现的，也就了解了 diff 算法。
* patch(container, vnode);
* patch(vnode, newVnode);

```javascript
// patch(container, vnode);
function createElement(vnode) {
    var tag = vnode.tag  // 'ul'
    var attrs = vnode.attrs || {}
    var children = vnode.children || []
    if (!tag) {
        return null
    }

    // 创建真实的 DOM 元素
    var elem = document.createElement(tag)
    // 属性
    var attrName
    for (attrName in attrs) {
        if (attrs.hasOwnProperty(attrName)) {
            // 给 elem 添加属性
            elem.setAttribute(attrName, attrs[attrName])
        }
    }
    // 子元素
    children.forEach(function (childVnode) {
        // 给 elem 添加子元素
        elem.appendChild(createElement(childVnode))  // 递归
    })

    // 返回真实的 DOM 元素
    return elem
}
```
```javascript
// patch(vnode, newVnode);
function updateChildren(vnode, newVnode) {
    var children = vnode.children || []
    var newChildren = newVnode.children || []

    children.forEach(function (childVnode, index) {
        var newChildVnode = newChildren[index]
        if (childVnode.tag === newChildVnode.tag) {
            // 深层次对比，递归
            updateChildren(childVnode, newChildVnode)
        } else {
            // 替换
            replaceNode(childVnode, newChildVnode)
        }
    })
}

function replaceNode(vnode, newVnode) {
    var elem = vnode.elem  // 真实的 DOM 节点
    var newElem = createElement(newVnode)

    // 替换
}
```
上面的 createElement、updateChildren 仅仅是对 DOM 元素做了最简单的对比，就像本节开始提醒到的，我们现在在了解的阶段，无需去关注细节，把握大体实现流程即可。本文没有涉及到的内容，比如节点的新增和删除、节点的重新排序、节点的样式、属性、事件绑定等内容，如果有兴趣的同学可以自己下来慢慢研究。

以上就是 virtual DOM 与 diff 算法入门介绍的全部内容了，我们从为什么会有 virtual DOM 入手，介绍了它是什么以及如何应用，同时介绍了最核心的 diff 算法，希望对大家有所帮助。
