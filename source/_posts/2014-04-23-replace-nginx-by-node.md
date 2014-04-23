title: 用 node 代替 nginx 做请求分发
date: 2014-04-23 22:18:30
tags: node
---

对于一个大型 Web 网站来说，网站往往是由多套代码来提供服务，并通过 nginx 等 HTTP Server 把请求分发到各个 Web 服务器上。例如:

```
                      +–––––––––––––––––+
               /a     |                 |
      +–––––––––––––––+  10.0.1.1:8080  |
      |               |                 |
      |               +–––––––––––––––––+
      |
+–––––+–––+           +–––––––––––––––––+
|         |    /b     |                 |
|  nginx  +–––––––––––+  10.0.1.2:8081  |
|         |           |                 |
+–––––+–––+           +–––––––––––––––––+
      |
      |               +–––––––––––––––––+
      |        /c     |                 |
      +–––––––––––––––+  10.0.1.3:8082  |
                      |                 |
                      +–––––––––––––––––+
```

这样在本地开发的时候，开发者为了搭建一个完整的开发环境，就需要装一个 nginx。团队人多了以后，nginx 的规则同步是个让人头疼的事情，特别是配置越来越复杂的时候，每个新同事进来都要折腾一下。

## node proxy 代替 nginx

如果能用 node 实现 nginx 的请求分发功能，那就可以把整个请求分发服务提交到代码仓库，分发规则修改后团队成员就只需要更新本地代码即可。

Google 了一下找到了 [node-http-proxy] 这个模块，可以实现 http 代理功能。用 node-http-proxy 可以很容易的实现 nginx 的请求分发功能:

```javascript
var servers = {
    a: '10.0.1.1:8080',
    b: '10.0.1.2:8081',
    c: '10.0.1.3:8082'
};

var proxy = require('http-proxy').createProxyServer(); // proxy 服务
var http = require('http');
http.createServer(function (req, res) {
    var target = null;

    // 根据 url 设置目标服务地址
    if (req.url.indexOf('/a') === 0) {
        target = servers.a;
    } else if (req.url.indexOf('/b') === 0) {
        target = servers.b;
    } else {
        target = servers.c;
    }

    // 分发请求
    proxy.web(req, res, {
        target: c
    });
}).listen(8000);
```

[node-http-proxy]: https://github.com/nodejitsu/node-http-proxy
