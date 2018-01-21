---
title: WebSocket介绍以及配合STOMP的使用
date: 2018-01-21 01:09:54
tags: WebSocket, STOMP
---

由于近期需要使用WebSocket的部分功能，然而在工作过程中，发现自己对这部分知识点不是很了解，而且对于后台同学提出的WebSocket和STOMP的组合，不知如何下手。经过相关资料查证，分享与大家，如有纰漏，希望不吝指出。  
本文行文为三个部分，分别讲述：Socket是什么，WebSocket是什么，STOMP是什么，如何结合后两者投入使用。  

## 1. Socket 
目前来说，我们经常说的Socket的有好几种意思，而且这几种意思还都与通信有关，他们分别是： 
1. Socket连接  
socket连接，是端到端的一种连接方式，连接上之后，双方可以互发数据，完成交互；socket连接的建立也是一个三次握手的过程，经过这个过程之后，双方都可以通过事件监听来获取来自对方的消息(connect, data, close ...）,也可以主动发送消息给对方（Socket.write）。Socket连接在不同语言的网络模块均有提供，以上方法都是node的net模块提供的一些方法和事件，可以用来建立一个完整的socket连接。
2. Socket抽象封装层 
这一种意思是说，它是作为我们所说的网络分层结构里面的，网络层和应用层之间的一层抽象封装。它的作用，就是将功能强大的网络层的操作做了一个封装，将其复杂的操作，抽象为几个简单的接口供应用层调用，以实现进程在网络中通信。按照网络上流行的说法，TCP/IP（网络层）是功能强大的发动机引擎，Socket层是汽车，我们只需要动动方向盘，就能调动起强大的引擎为我所用。
3. 套接字
这个部分，说的是Socket连接建立起来之后，双方维护的一个对象，用来发送和接受数据包。一个Socket连接建立，对应的是连接两端对应的一对套接字对象，其维护的信息为：连接使用的协议，本地主机的IP地址，本地进程的协议端口，远地主机的IP地址，远地进程的协议端口。通过如上信息，即可确定传输的位置和传输的方式。  

## 2. WebSocket 
1. 是什么  
WebSocket是H5规范提出的一种应用层协议（与HTTP处于同一层级），是建立在TCP/IP协议族之上的一种长连接，可进行全双工通信。  
2. 为什么需要它  
它的提出确实是极其必要的。主要有两方面的考虑：一是，在H5规范的描述下，web应该是一个丰富多彩的世界，能提供应用程序级别的使用体验。既然是应用程序级别体验，自然应该有应用程序级别的网络基础支持，而这种支持就应该包含长连接，实时通信这种级别的支持；二是，使用目前的HTTP协议，模拟出两端长连接的效果（轮询，阻塞），消耗太大。
3. 实现的过程  
WebSocket连接实现的过程分为两个部分：建立连接的过程，连接之后的Socket通信过程。  
WebSocket连接建立的过程，是用到了HTTP请求的。在一开始建立连接的过程中，希望建立连接的客户端会向服务端发送一个HTTP请求，询问服务器是不是支持WebSocket，并且告诉服务端，我使用WebSocket请求，希望服务端进行相应的响应。  
此处为了区分普通的HTTP请求，此处上传了其他的头部信息：  
```
// 随机生产的Base64字符串，用于安全校验
Sec-WebSocket-Key：n0seXxkGzvDzqsH8ZkDfcg==
// 指定子协议和版本号
Sec-WebSocket-Protocol：v10.stomp, v11.stomp
Sec-WebSocket-Version：13
// 请求服务器升级为WebSocket
Connection:Upgrade
Upgrade：WebSocket

// 服务端的回应
Connection:Upgrade
Sec-WebSocket-Accept:8BiqnztuCvGwd9ine9abKXjtzE0=
Sec-WebSocket-Protocol:v10.stomp
Upgrade:WebSocket
```
  在客户端校验Sec-WebSocket-Accept通过之后，连接即可建立完成。这之后的信息通讯均是WebSocket定义的通过长连接进行的，而且此长连接会复用刚才HTTP请求建立的tcp长连接。之后的消息发送，消息接受，连接建立，连接关闭等交互，与Socket基本类似。  
4. 如何使用node搭建一个简单的ws服务器  
此处的demo是，通过sockjs，建立一个ws服务器，连接两个或者多个客户端，当某一个客户端发送消息给服务器，服务器可以主动将该消息发送给别的客户端。
```
// 服务端主要代码
var http = require('http');
var sockjs = require('sockjs');

// 建立socket连接
var sockjs_opts = {sockjs_url: "http://cdn.jsdelivr.net/sockjs/1.0.1/sockjs.min.js"};

var sockjs_echo = sockjs.createServer(sockjs_opts);
sockjs_echo.on('connection', function(conn) {
    conn.on('data', function(message) {
        conn.write(message);
    });
});

sockjs_echo.installHandlers(server, {prefix:'/echo'});
server.listen(9999, '0.0.0.0');
```

 ```
// 客户端主要代码
var sockjs_url = '/echo';
var sockjs = new SockJS(sockjs_url);

sockjs.onopen    = function()  {print('[*] open', sockjs.protocol);};
sockjs.onmessage = function(e) {print('[.] message', e.data);};
sockjs.onclose   = function()  {print('[*] close');};

// 产生交互信息
sockjs.send(‘some message’);
```
点击查看[sockjs官方完整demo](https://github.com/chirsen/sockjs-node/tree/master/examples)


## 3. STOMP 
Simple (or Streaming) Text Orientated Messaging Protocol，简单(流)文本定向消息协议，它提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理（Broker）进行交互。 简单来说，就好像HTTP定义了TCP的相关细节一样，STOMP在WebSocket协议之上，告诉信息交互的双方，消息的格式是什么，应该怎样收发的文本协议。具体的定义内容为：
STOMP是基于frame（帧）的协议，每个frame都包含了一个command，一系列的可选headers和消息本身的body，如下：
```
COMMAND
header1:value1
header2:value2

Body^@
```
 上面的空行部分必需，分割headers和body。除了上述的帧内容的定义，协议还对不同的操作定义了不同COMMAND的帧。
 ```
 // 客户端：
 SEND			// 发送消息到服务端，可添加自定义的header，body携带内容
 SUBSCRIBE		// 用于注册给定目的地send帧，被注册的目的地收到任何消息豆浆通过MESSAGE帧发送过来
 UNSUBSCRIBE	// 取消注册监听
 BEGIN			// 事务操作开始
 COMMIT			// 事务提交
 ABORT			// 事务过程中的回滚
 ACK			// 确认订阅消息的消费
 NACK			// NACK有ACK相反地作用。它地作用是告诉server client不想消费这个消息
 DISCONNECT		// 断开连接
 // 服务端
 CONNECT		// 连接建立
 RECEIPT		// server成功处理请求带有receipt的client frame后的返回
 ERROR			// 如果出错的话，server将发送ERRORframe
 MESSAGE		// 将订阅的消息发送给client
 ```
更多命令详解，可参考<a href="http://blog.csdn.net/joeysheng/article/details/51970233">STOMP协议参考</a>

## 4. 结合使用 
在了解了上诉两个协议之后，我们需要把两方结合起来，让WebSocket消息操作变得规范，可控，易于理解。因为STOMP协议和WebSocket都有已经实现了且可靠的库，在这里我们直接采用。WebSocket采用sockjs，STOMP采用stompjs。
```
// 服务端主要代码：
var http = require("http");
var StompServer = require('stomp-broker-js');

var server = http.createServer();

server.listen(61614);

var stompServer = new StompServer({
  server: server,
  path: '/stomp'
});
// 将监听的客户端放入列表中，方便服务端在接受到消息之后进行转发
stompServer.on('connected', function(sessionId, headers){
  var clientId = headers.clientId;
  if(clientId) {
    clients.push(clientId);
  }
});

stompServer.on('error', function(error) {
  // 将订阅对象减少一个(错误对象)
  clients.splice(clients.length - 1, 1);
  return;
});

// 在每次对应的roomid频道收到消息时，转发给所有的订阅者
stompServer.subscribe(config.desination, function(msg, headers) {
  for(var i = 0; i < clients.length; i++) {
    // 如果时debug，打印每次的请求和消息
    if(config.debug) {
      console.log('header: \n' + headers);
      console.log('msg: \n' + msg);
    }
    // 消息转发
    stompServer.send(clients[i], {'content-type': 'application/json'}, JSON.stringify({
      headers: headers,
	  msg: msg
    }));
  }
});
```
此处的服务端代码，是直接传入创建的server，即可使得server支持STOMP协议。其实在这一步时做了很多工作。其中就有，调用stompjs库，将sockjs的消息发送用stomp进行改写，将WebSocket的方法统统用STOMP协议的方法进行了包装一遍。这里举消息包装和方法包装的例子说明。
```
// 当调用wensocket的send方法的时候
this.send = function (topic, headers, body) {
  // 将消息内容组装成stomp协议的一帧
  var _headers = {};
  if (headers) {
    for (var key in headers) {
      _headers[key] = headers[key];
    }
  }
  var frame = {
    body: body,
    headers: _headers
  };
  var args = {
    dest: topic,
    frame: this.frameParser(frame)
  };
  this.onSend(selfSocket, args);
}

this.onSend = function (socket, args, callback) {
  ...
  this.emit('send', {frame: {headers: frame.headers, body: bodyObj}, dest: args.dest});
  // 将消息发送给订阅方
  this._sendToSubscriptions(socket, args);
  if (callback) {
    callback(true);
  }
};

this._sendToSubscriptions = function (socket, args) {
  ...
  // 确定订阅方，凭借上command，进行发送
  args.frame.command = "MESSAGE";
  var sock = sub.socket;
  if (sock !== undefined) {
    stomp.StompUtils.sendFrame(sock, args.frame);
  } else {
    this.emit(sub.id, args.frame.body, args.frame.headers);
  }
};
// 发送的时候，还是采用WebSocket的发送
function sendFrame(socket, _frame) {
  var frame_str = null;
  if (!_frame.hasOwnProperty('toString')) {
    var frame = new Frame({
      'command': _frame.command,
      'headers': _frame.headers,
      'body': _frame.body
    });
    frame_str = frame.toString();
  } else {
    frame_str = _frame.toString();
  }
  // WebSocket发送
  socket.send(frame_str);
  return true;
}

```


```
// 客户端主要代码:
var url = "ws://localhost:61614/stomp";
var client = Stomp.client(url);

function afterConnect(roomid) {
  btn.addEventListener('click', function () {
    var msg = input.value;
    // 发送信息
    client.send(roomid, {}, msg);
  }, false);
}

function createConnect(roomid, uid) {
  client.connect(headers, function (error) {
    if (error.command == "ERROR") {
      console.error(error.headers.message);
    } else {
      afterConnect(roomid);
      // 订阅自己的客户端id，方便收到服务器发送过来的信息
      client.subscribe(uid, function (msg) {
        var body = msg.body;
        if (msg.headers['content-type'] == 'application/json') {
          body = JSON.parse(msg.body)
        }
      });
    }
  });
}
```
点击查看[完整demo](https://github.com/chirsen/stomp-ws-server)

## 总结
在各方面了解完WebSocket和STOMP相关内容之后，其实我们可以发现，STOMP是个很简单的协议，但是这个简单协议却能有效的规约前后端的交互过程，使交互过程清晰有效。这种用简单高效的抽象，完成通用复杂的工作的方法，其实是很值得我们去借鉴的。另外，在完成这部分内容的探索学习过程中，还顺便学习了一下npm包发布的相关内容。感觉学习新东西确实是总能给人带来益处，大家加油！
