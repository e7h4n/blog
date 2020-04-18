title: Express 4.0 升级手记
date: 2014-04-24 17:15:01
tags: node
---

前几天著名的 Web 框架 [express] 更新到了 4.0 版，这个版本带来了一些比较大的更新，并且不再向下兼容 3.x 版。今天花了一点时间把项目升级到了 4.0 版，在此记录以下遇到的问题。

<!-- more -->

## Express 4.0 不再内置 connect 中间件

这是影响比较大的一个修改，原先的各种 connect 中间件都需要单独 `require` 再使用。以下是修改前后的中间件对应关系:

``` javascript
express.favicon        === require('static-favicon')
express.logger         === require('morgan')
express.json           === require('body-parser').json
express.urlencoded     === require('body-parser').urlencoded
express.methodOverride === require('method-override')
express.static         === require('serve-static')
```

看起来复杂，实际上只要做个字符串替换就成了。完整列表参考 [connect 中间件列表]。

## 不再需要 `app.use(app.router)`

影响第二大的更新，express 3.x 中，`app.get` 方法插入中间件的执行时机是由 `app.use(app.router)` 的位置决定的，例如:

```javascript
app.use(app.router);
app.use(function (req, res, next) {
    console.log('foo');
    next();
});
app.get('/', function (req, res, next) {
    console.log('bar');
    res.send(200);
});
```

上面代码在 express 3.x 中的 console 输出是:

```
bar
foo
```

Express 4.0 不需要再显式引入 `app.router`，因此上面代码删除掉第一行后，在 express 4.0 中的 console 输出是:

```
foo
bar
```

升级到 4.0 不仅仅是简单的删除 `app.use(app.router)` 就行了，一定要仔细检查代码会不会受到中间件执行顺序的影响。

## 删除了 `app.configure`

下面两段代码在 express 3.0 中的作用完全相同，express 推荐使用第二种写法，第一种写法在 4.0 中被取消了。

```javascript
// 写法一
app.configure('development', function () {
    // ...
});

// 写法二
if ('development' === app.get('env')) {
    // ...
}
```

## 删除了 `express.createServer`

直接用 `express()` 来代替 `express.createServer()`

## 其他

其余还有一些细节上的修改，例如移除了 `res.on('header')`、`res.charset`，将 `res.headerSent` 改为 `res.headersSent` 等等。可以查看详细的升级教程 [Migrating from 3.x to 4.x]。

参考文献:

* [connect 中间件列表]
* [Migrating from 3.x to 4.x]

[connect 中间件列表]: https://github.com/senchalabs/connect?source=c#middleware
[Migrating from 3.x to 4.x]: https://github.com/visionmedia/express/wiki/Migrating%20from%203.x%20to%204.x
