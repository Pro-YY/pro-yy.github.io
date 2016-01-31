---
layout: post
title: "Session with Express and Nginx"
date: 2015-07-17 17:12:56 +0800
comments: true
categories:
---

## 为什么一定要持久化存储会话信息

如果会话信息仅仅存储在内存...

- 服务器重启，用户将会登出
- 无法在多个node进程(或集群环境)中共享会话

所以，会话(无论是HTTP会话还是Web-Socket会话)都需要在服务器端持久化存储在数据库中，如Redis/Mongo。

环境： Express 4.12.2， Nginx 1.8.0

## Express添加Redis会话存储

需要的相关库或组件：

- [connect-redis](https://www.npmjs.com/package/connect-redis): Redis会话存储。
- [socket.io-redis](https://www.npmjs.com/package/socket.io-redis): 多个socket.io进程实例可以彼此触发/广播事件。

{% codeblock lang:javascript app.js %}
var http = require('http');
var socketio = require('socket.io');
var session = require('express-session');           // 会话支持
var passport = require('passport');                 // 身份认证，依赖express-session
var RedisStore = require('connect-redis')(session); // redis存储，用于持久化会话
var RedisAdapter = require('socket.io-redis');      // 存储web-socket的连接
var mongoose = require('mongoose');
// ...
var app = express();
var server = http.Server(app);
var sio = socketio(server);
{% endcodeblock %}

配置Redis会话存储HTTP会话
{% codeblock lang:javascript app.js %}
var sessionMiddleware = session({
  name: 'XXX:sess',
  secret: 'XXX',
  proxy: true,
  resave: false,
  store: new RedisStore(),
  saveUninitialized: false,
  cookie: {
    //secure: true, // only https
    maxAge: 7*24*60*60*1000 // 1 week
  }
});
// 应用会话存储中间件
app.use(sessionMiddleware);
// passport登录认证
app.use(passport.initialize());
app.use(passport.session());
{% endcodeblock %}

配置Redis会话存储Web-Socket会话
{% codeblock lang:javascript app.js %}
sio.adapter(RedisAdapter({host: 'localhost', port: 6379}));
// 为socket server添加redis存储的中间件
sio.use(function(socket, next) {
  sessionMiddleware(socket.request, socket.request.res, next);
});
// 对未登录的用户禁用socket支持
sio.use(function(socket, next){
  var passport = socket.request.session.passport;
  if (!passport || !passport.user) {
    logger.debug('no socket support without login');
    next(new Error('Authentication error'));
  }
  else next();
});
{% endcodeblock %}


## Nginx配置支持负载均衡

Web-Socket的连接建立需要同一个tcp连接上的多次握手支持，要求比一般的HTTP会话高一些。
所以为实现多进程的共享会话支持，需要配置支持iphash的upstream。

```
upstream nodes {
  ip_hash;
  server localhost:8001;
  server localhost:8002;
  server localhost:8003;
  server localhost:8004;
  keepalive 512;
}

server{
  listen 443 ssl;
  server_name example.com;
  location / {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_pass http://nodes;
  }
}
```
注意，可以用pm2启动多个node服务进程，但必须以fork mode启动，而不是-i的集群方式启动，负载均衡完全交给nginx即可。
如此：
```
pm2 start -f --name web1 app.js -- 8001
pm2 start -f --name web2 app.js -- 8002
pm2 start -f --name web3 app.js -- 8003
pm2 start -f --name web4 app.js -- 8004
```

## 参考资料
[Socket.io doc: using multiple nodes](http://socket.io/docs/using-multiple-nodes/)
