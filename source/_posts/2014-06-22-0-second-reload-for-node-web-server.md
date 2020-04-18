title: 平滑升级 Node Web 服务器
date: 2014-06-22 11:33:12
tags: node
---

用 Node 搭建一个 Web 服务是一件很轻松的事情，例如经典的 Hello World 例子，三行代码实现一个 Web 服务:

```javascript
require('http').createServer(function (req, res) {
    res.end('Hello World');
}).listen(3000);
```

但是，要做一个稳定可靠的线上 Web 服务，并不简单。例如[异常处理]、日志、部署、服务更新等等。本文主要讨论 Node Web 服务的更新。

<!-- more -->

## 自然停机

在 Web 服务更新的过程中，有两个主要步骤，停止旧服务以及启动新服务。如果不能妥善的停止旧服务，那么对于已经在使用旧服务的用户，就会看到一个出错页面。比如这个 Bad Case:

```javascript
require('http').createServer(function (req, res) {
    setTimeout(function () {
        res.end('Hello World');
    }, 10000);
}).listen(3000);
```

代码中的 `setTimeout` 用来模拟一个耗时请求。启动这个服务，随后在浏览器中访问 `http://localhost:3000/`，在等待响应的过程中，使用 kill 强行停止服务，浏览器会立即给出一个 `No Data Received` 页面。这是由于 Node 在收到 kill 发出的 TERM 信号后会立即退出进程并关闭所有已经建立的链接。如果这是一个线上服务，那么在更新服务的一瞬间，已经建立连接的用户就会看到一个 `No Data Received` 页面，这是一种非常粗暴的做法。

为了解决这个问题，Node 自身的 `net.Server` 模块提供了一个 `close` 方法:

> server.close([callback])
>
> Stops the server from accepting new connections and keeps existing connections. This function is asynchronous, the server is finally closed when all connections are ended and the server emits a 'close' event. Optionally, you can pass a callback to listen for the 'close' event.

这个方法在调用时会停止接收新的连接请求，但不会立即关闭已经建立的连接，而是会等待这些连接自然结束。一个简单的例子:

```javascript
process.on('SIGQUIT', function () {
    // 判断是否有正在进行的连接
    if (!server.getConnections()) {
        process.exit(0);
    }

    console.log('等待连接结束');
    server.close(function () {
        console.log('All');
    });
});
```

运行代码，浏览器中访问一下，然后通过 `kill -QUIT PID` 来终止进程。可以看到进程不会立即退出，而是等到浏览器中返回结果后才退出。

在实际应用中，`server.close` 方法往往要等待很久才会退出，这个问题有很多原因，例如服务器代码错误、浏览器 Keep Alive 保持了额外的连接等等。对于这类问题，可以设置一个超时时间，若服务器在一段时间之后仍然有连接不能释放，就强行退出服务:

```javascript
process.on('SIGQUIT', function () {
    // 判断是否有正在进行的连接
    if (!server.getConnections()) {
        process.exit(0);
    }

    console.log('等待连接结束');
    server.close(function () {
        console.log('All');
    });

    // 15 秒后仍然不能关闭所有连接的话就直接停止进程
    setTimeout(function () {
        process.exit(0);
    }, 15000);
});
```

## 无缝切换

旧服务的关闭和新服务的启动之间必然有一个无服务的时间间隔，无缝切换指的是在这个时间间隔内程序依然能正常服务。常见的做法是多机部署 + nginx 的负载均衡模块。更新服务时只需要逐台部署，保证同一时刻至少有一台机器在提供服务，nginx 就会将流量自动分配到正常服务的机器上。

其实，Node 本身的 `cluster` 模块可以在单机部署的情况下实现无缝切换，基本原理如下:

 1. 发一个重启信号给 Master，例如 `kill -USR2 MASTER_PID`
 2. Master 起 n 个新的服务，开始监听请求
 3. Master 停止原先旧服务的监听，并等待旧服务的所有连接结束
 4. 关闭旧服务

完整代码:

```javascript
var cluster = require('cluster');
var http = require('http');

if (cluster.isMaster) {
    cluster.fork();

    cluster.on('exit', function(worker, code, signal) {
        console.log('worker ' + worker.process.pid + ' 退出');
    });

    cluster.on('listening', function(worker, code, signal) {
        console.log('worker ' + worker.process.pid + ' 开始服务');
    });

    cluster.on('disconnect', function(worker, code, signal) {
        console.log('worker ' + worker.process.pid + ' 停止服务');
    });

    process.on('SIGUSR2', function () {
        // 保存旧 worker 的列表，cluster.workers 是个 map
        var oldWorkers = Object.keys(cluster.workers).map(function (idx) {
            return cluster.workers[idx];
        });

        // 起新服务
        cluster.fork();

        // 当新服务起起来之后，关闭所有的旧 worker
        cluster.once('listening', function (worker) {
            oldWorkers.forEach(function (worker) {
                // disconnect 会停止接收新请求，等待旧请求结束后再结束进程
                worker.disconnect();
            });
        });
    });
} else {
    http.createServer(function(req, res) {
        // 模拟慢速请求
        setTimeout(function () {
            res.writeHead(200);
            res.end("hello world\n");
        }, 15000);
    }).listen(8000);
}
```

在命令行中发送 `kill -USR2 MASTER_PID` 信号，可以看到整个更新的过程:

```bash
worker 370 开始服务
# 发送 USR2 信号
worker 422 开始服务
worker 370 停止服务
worker 370 退出
```

## 使用 pm2 来更新服务

[pm2] 是个强大的 Node 服务管理工具，其自带了负载均衡、服务管理、服务监控等多种功能。例如上文介绍的自然停机、无缝切换，使用 pm2 可以直接实现，不需要额外的开发工作。

pm2 重启服务有三个命令，分别是 `restart`, `reload` 以及 `gracefulReload`，具体的区别是:

`restart`: 直接关闭旧服务然后启动新服务，会造成已建立的连接失效

`reload`: 平滑更新，先启动若干个新服务，同时停止旧服务接收请求。等待旧服务都停止服务后，关闭旧服务。和上面 cluster 的代码原理类似，有可能因为要等待连接关闭造成重启时间比较长。

`gracefulReload`: 平滑更新，和 `reload` 的区别是 `gracefulReload` 会发送一个 `shutdown` 消息给旧服务，具体的停服逻辑可以由程序自己实现，比较灵活，例如:

```javascript
process.on('shutdown', function () {
    server.close();

    // 15 秒后仍然不能关闭所有连接的话就直接停止进程
    setTimeout(function () {
        process.exit(0);
    }, 15000);
});
```

## 本文小节

本文讨论了 Node Web 服务平滑更新的一些实践，具体技术点上有自然停机和无缝切换两个部分，核心技术并不复杂，但是细节较多，容易遗漏。pm2 提供了一整套平滑更新的方案供使用，目前 pm2 在我的团队中应用比较广泛，并且已经在线上环境中运行了比较久的时间，是一个不错的选择。

[异常处理]: http://lostjs.com/2014/01/25/handle-exception-in-node/
[pm2]: https://github.com/Unitech/pm2
