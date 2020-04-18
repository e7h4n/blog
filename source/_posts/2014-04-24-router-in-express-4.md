title: 浅谈 Express 4.0 Router 模块
date: 2014-04-24 22:02:45
tags: node
---

Express 是目前 node 社区最主要的 Web 框架，前不久刚刚升级到了 4.0 版本。与 3.x 版本比，4.0 版本拥有一个全新设计的 Router 模块，开发者可以更方便的对 middleware 进行隔离与重用。

<!-- more -->

## Express 3.x 时代的中间件 (middleware) 与控制器 (controller)

在 express 3.x 版本中，一个控制器往往不是业务逻辑的全部，中间件才是业务逻辑的大头。例如一个处理用户订单的服务，往往验证用户权限、读写数据库等主要逻辑工作都在中间件中就完成了，而控制器所做的大部分工作就是拼数据给视图 (view）。

一个标准的 URL 映射写法如下:

```javascript app.js
// 这里是一些中间件，express 的常见写法
var A = require('./middlewares/A');
var B = require('./middlewares/B');
var C = require('./middlewares/C');
app.get('/books', A, B, C, require('./controllers/book').index);
```

```javascript controllers/book.js
exports.index = function (req, res) {
    var retA = req.A; // 中间件 A 的输出结果
    var retB = req.B; // 中间件 B 的输出结果
    var retC = req.C; // 中间件 C 的输出结果
    // ... 其余程序逻辑
}
// ...
```

控制器与中间件的相互依赖关系是完全隐式的，不能通过代码分析来得到任何的保证。中间件的插入与控制器的代码被分离在了 `app.js` 与 `controllers/book.js` 两个文件中，不仅阅读起来不直观，修改起来也很容易出错。假如有一天团队的新同事修改了 `book.js` 去掉了中间件 `C` 的逻辑，但是忘记了修改 `app.js`（这是一个非常容易犯的错误），那么 express 不会报任何错误，code review 也很难发现，因为这两处代码离得实在是太远了。

这种设计同时会导致测试非常困难，因为单独 `require` 一个控制器是毫无意义的，因为控制器本身不可能独立于中间件来执行。如果要测试控制器，要么在单元测试代码在控制器前里现场装配中间件，要么就 mock 请求通过中间件后的数据。无论哪种做法，都需要单元测试完全了解中间件与控制器的业务逻辑才可能实现，这样就增大了单元测试的难度。

一个解决方法是将二者的依赖关系倒置，把控制器模块写成一个接收 `app` 作为参数的函数，在函数内部装配中间件与控制器。代码如下:

```javascript app.js
require('./controllers/book')(app);
```

```javascript controllers/book.js
var A = require('../middlewares/A');
var B = require('../middlewares/B');
var C = require('../middlewares/C');

module.exports = function (app) {
    app.get('/books', A, B, C, function (req, res) {
        var retA = req.A; // 中间件 A 的输出结果
        var retB = req.B; // 中间件 B 的输出结果
        var retC = req.C; // 中间件 C 的输出结果
        // ... 其余程序逻辑
    });
};
// ...
```

这样做虽然提高了代码的内聚性，但是直接把 `app` 暴露给其它模块使得 `app` 有被滥用的风险，让二者从面向接口的松散耦合变成了直接操纵实例的强耦合。同时这种方案不仅没有提高可测试性，反而大大提高了单元测试的难度（想想现在都需要 mock 一个 `app` 了 T_T）。

为了更好的解决这个问题，Express 4.0 给出了更好的解决方式: `express.Router`。


## 使用 `express.Router` 来组织控制器与中间件

`express.Router` 可以认为是一个微型的只用来处理中间件与控制器的 `app`，它拥有和 `app` 类似的方法，例如 `get`、`post`、`all`、`use` 等等。上面的例子使用 `express.Router` 可以修改为:

```javascript app.js
app.use(require('./controllers/book'));
```

```javascript controllers/book.js
var router = require('express').Rouer(); // 新建一个 router

var A = require('../middlewares/A');
var B = require('../middlewares/B');
var C = require('../middlewares/C');

// 在 router 上装备控制器与中间件
router.get('/books', A, B, C, function (req, res) {
    var retA = req.A; // 中间件 A 的输出结果
    var retB = req.B; // 中间件 B 的输出结果
    var retC = req.C; // 中间件 C 的输出结果
    // ... 其余程序逻辑
});

// ...

// 返回 router 供 app 使用
module.exports = router;
```

通过 `express.Router`，控制器与中间件的代码紧密的联系在一起，并且避免了传递 `app` 的潜在风险。同时，一个 `router` 就是一个完整的功能模块，不需要任何装配就可以执行。这一点对于单元测试来说非常简单。

## `express.Router` 的其他特性

### 中间件重用

上面提到过，`express.Router` 可以认为是一个迷你的 `app`，它拥有一个独立的中间件队列。这个特性可以用来共享一些常用的中间件，例如:

```javascript express 3.x
// parseBook 是个中间件
app.get('/books/:bookId', parseBook, viewBook);
app.get('/books/:bookId/edit', parseBook, editBook);
app.get('/books/:bookId/move', parseBook, moveBook);

app.get('/other_link', otherController);
```

Express 4.0 的写法:

```javascript express 4.0
var bookRouter = express.Router();
app.use('/books', bookRouter);

bookRouter.use(parseBook);
// 下面三个控制器都会经过 parseBook 中间件
bookRouter.get('/books/:bookId', viewBook);
bookRouter.get('/books/:bookId/edit', editBook);
bookRouter.get('/books/:bookId/move', moveBook);

app.get('/other_link', otherController); // 不会经过 parseBook 中间件
```

这个例子中 `bookRouter` 使 `parseBook` 这个中间件得到了充分的重用。

### 搭建 rest-ful 服务

Code talks:

```javascript
var bookRouter = express.Router();
bookRouter
    .route('/books/:bookId?')
    .get(function (req, res) {
        // ...
    })
    .put(function (req, res) {
        // ...
    })
    .post(function (req, res) {
        // ...
    })
    .delete(function (req, res) {
        // ...
    })
```

## 小节

`express.Router` 是 express 4.0 中我最喜欢的更新，我认为 express 4.0 的代码里应该尽量多使用 `express.Router` 来代替原先的 `app.get` 方式。原先 URL 路径、中间件、控制器三者的松散关系可以借由 `express.Router` 变得紧密，整个控制器变成了一个不依赖于任何外部实例的独立模块，更有利于模块的拆分（想想把网站的各个模块都拆成独立的 router 吧），同时对于测试也更加友好。
