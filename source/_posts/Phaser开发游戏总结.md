---
title: Phaser开发游戏总结
date: 2018-01-08 18:13:33
tags: 游戏
---

## 前言

Phaser是一个非常好用的html5游戏开发框架，官网上是这样介绍的：“一个快速、免费并且完全开源的框架，提供Canvas和WebGL两种渲染方式，致力于增强桌面端与移动端浏览器游戏的体验”。

## 开始

### 开始游戏的场景

html5标准新加了一个 canvas标签，在canvas上我们可以通过js绘制各种各样的内容，游戏内包含着场景，精灵等要素。我们绘制了宽度600高度250，的一个canvas画布。游戏渲染模式使用Phaser.AUTO，也就是自动检测，在浏览器支持WebGL的时候使用WebGL渲染，不支持的时候回退到Canvas渲染。。并且加载了Splash场景，通过start，进入了Splash场景。等Splash场景结束后，我们可以通过`game.state.start('Main');`来加载Main场景实现场景之前的切换。在场景中有各种各样的方法来控制场景的展示，init方法，preload方法，create方法和update方法，分别管理当前场景的初始化、预加载、生成游戏对象以及更新游戏循环。

```javascript  
const game = new Phaser.Game(600, 250 , Phaser.AUTO,"");
const main = new Phaser.State();
game.state.add('Splash', Splash);
game.state.add('Main', Main);
game.state.start('Splash');
```

通过这些方法，就可以完成一个非常炫酷的Phaser游戏了

### 丰富我们的游戏

初始化Init方法：启动物理引擎（ARCADE），这是Phaser框架自带的最简单的物理引擎，用于矩形盒的碰撞检测。。
```javascript 
main.init = function(){
    game.physics.startSystem(Phaser.Physics.ARCADE);
    game.world.enableBody = true;
}
```

预加载方法：加载各类游戏资源，并设置唯一id，被精灵引用。

```javascript  
game.load.image('floor', 'img/floor.png');
```

生成游戏对象方法：生成游戏地图

```javascript  
main.create = function(){
    this.floors = game.add.group();
    ...
    //floor
    for(var i=1; i < n; i++){
        var floor = game.add.sprite(30*i, 90, 'box','', this.floors);
    }
}
```

更新循环方法：通过方向键控制主角左右移动和跳跃，当主角撞到地板，做销毁处理，并且重新开始游戏。这样我们就完成一个简单的跳跃障碍物游戏。

```javascript  
main.update = function(){
    game.physics.arcade.overlap(this.player, this.floor, this.kill, null, this);
    ...
    if (this.cursor.left.isDown) 
        this.player.body.velocity.x = -200;
    ...
    if (this.cursor.up.isDown && this.player.body.touching.down) 
        this.player.body.velocity.y = -250;
}
```

## Phaser开发游戏问题总结

### iPhone下游戏显示模糊

这是因为iPhone现在都是retina屏幕，在retina屏幕下，会用2个像素点的宽度去渲染图片的1个像素点，因此该图片在retina屏幕上实际会占据200x200像素的空间，相当于图片被放大了一倍，因此图片会变得模糊。所以我们在初始化canvas大小不应该是屏幕的 大小去渲染，使用屏幕大小俩倍做渲染，同时通过css来讲canvas缩小，就可以解决问题。也可以通过dpi来做渲染相应大小。

```javascript  
const dpi = window.devicePixelRatio || 2;
const width = document.documentElement.clientWidth * dpi;
const height = document.documentElement.clientHeight * dpi;

class Game extends Phaser.Game {
    constructor() {
        super(width, height, Phaser.CANVAS, 'content', null);
    }
}
```

### 资源问题

Phaser社区版本提供了 grunt打包工具，可以自行缩减比如常用 wap端游戏不需要的按键控制，多余的物理引擎，来缩小资源大小。
整个资源打包也可以通过webpack内置的压缩进行优化。
游戏的图片其实对于整个资源占比很大，对一些按钮，icon，标志图片等较小图片可以进行合图操作，减少大量的http请求，对一些超过1024*1024大小的图片进行些许压缩。

### 内存优化

* 减少不必要的计算
    *  图片阴影，发光效果，添加mask效果，可以直接用图片替代
    *  复杂文字效果使用图片
* 游戏内不直接使用setTimeout setInterVal
* 精灵数量的控制和注意及时的销毁，保证内存不泄露
* 在主循环update逻辑做到`精简`，避免大片业务逻辑放到上面
* 动画不放到update里 比如位置移动，可以使用补间动画(tween)

```javascript  
update() {
    sprite.x += 1;
}
```

## Phaser 学习资源	

[Phaser插件合集](https://github.com/orange-games)	
[Phaser官网](http://www.phaser.io)	
[Phaser中文官网](http://phaserengine.com)	
[Phaser小游戏合集](https://github.com/channingbreeze/games)
[Phaser webpack配置](https://github.com/lean/phaser-es6-webpack)


