title: 异步、promise 与缓存
date: 2014-04-21 22:01
tags: [nodejs, promise]
---

## 异步操作缓存

在 Node 开发过程中要常常和异步调用打交道，例如发送 http 请求、读取文件内容等。有时我们需要将这些异步 I/O 操作的结果缓存下来，使得程序运行的速度更快，例如:

```javascript
var fileContent = null;
function readFile(callback) {
    if (fileContent !== null) {
        process.nextTick(function () {
            callback(null, fileContent);
        });
        return;
    }

    fs.readFile(FILE_PATH, function (err, content) {
        if (err) {
            callback(err);
            return;
        }

        fileContent = content;
        callback(null, content);
    });
}
```

上面的代码用 `fileContent` 变量做了一个简单的内存缓存，在 `readFile` 函数中，如果发现缓存中存在内容，则跳过文件读取操作。

这是个非常简单的缓存应用案例，我们可以将代码中的缓存逻辑抽出来，与业务逻辑分离，成为一个通用的缓存方法 `cacheAsync`:

```javascript
function cacheAsync(logical) {
    var cache = null;
    return function (callback) {
        if (cache) {
            process.nextTick(function () {
                callback(null, cache);
            });
            return;
        }

        logical(function (err, result) {
            if (err) {
                logical(err);
                return;
            }

            cache = result;
            callback(null, result);
        });
    };
}

var readFile = cacheAsync(function (callback) {
    fs.readFile(FILE_PATH, callback);
});

readFile(function (err, content) {
    if (err) {
        throw err;
        return;
    }

    console.log('file:', content);
});
```

使用 `cacheAsync` 方法可以很容易的给一个异步回调函数加上缓存。

## promise

`promise` 是和上面代码中示范的 callback 异步回调方式截然不同的另一种异步范式。`promise` 范式下的异步函数不再接收一个 callback 函数，而是返回一个 promise 对象。promise 对象通过 `then` 方法来绑定回调函数，通过 `catch` 方法绑定错误处理函数。

nodejs 目前有很多个 promise 异步范式的库，最流行的是 `q`。这里以 q 为例，示范一个简单的 promise 异步范式:

```javascript
var readFile = function () {
    // q.nfcall 使一个 callback 风格的异步函数返回一个 promise 对象
    return q.nfcall(fs.readFile, FILE_PATH);
};

// 下面这行看起来就和同步调用一样
var fileContent = readFile();

fileContent.then(function (content) {
    console.log('file:', content);
}).catch(function (err) {
    console.log(err);
});
```

看起来是不是感觉代码更复杂了？除了能看到异常逻辑和正常流程被分离了以外，似乎没有太多好处？那么看一个更复杂的例子，这个例子尝试从两个数据源读取用户信息:

```javascript
function getUserSync() {
    var user = fetchUserFromWeiboSync();

    if (!user) {
        user = fetchUserFromRenrenSync();
    }

    if (!user) {
        throw new Error(404);
    }

    return user;
}
```

callback 回调版本，一个常见的 callback hell:

```javascript
function getUserAsync(callback) {
    fetchUserFromWeiboAsync(function (err) {
        if (err) {
            fetchUserFromRenrenAsync(function (err, user) {
                if (err) {
                    callback(new Error(404));
                } else {
                    callback(null, user);
                }
            });
        } else {
            callback(null, user);
        }
    })
}
```

promise 版本，代码之少令人惊讶:

```javascript
function getUserQ() {
    return fetchUserFromWeiboQ().catch(fetchUserFromRenrenQ);
}
```

## promise 与缓存

给一个 promise 范式下的异步函数加缓存是一件非常轻松的事情，因为你不需要处理任何的异常流程:

```javascript
var q = require('q');

function cacheQ(logical) {
    var cache = null;

    return function () {
        if (cache !== null) {
            return q(cache);
        }

        // 这里不需要再处理任何的异常，缓存方法和程序逻辑进一步解耦
        return logical().then(function (result) {
            cache = result;
            // 这里必须 return，因为这里的返回值会作为参数传递给下一个 then 方法绑定的回调函数
            return result;
        });
    };
}

var readFile = cacheQ(function () {
    // q.nfcall 使一个 callback 风格的异步函数返回一个 promise 对象
    return q.nfcall(fs.readFile, FILE_PATH);
});
```

## 更灵活的缓存 

在实际应用中，一个缓存函数还应当支持以下特性:

1. 缓存应该是一个 key-value 存储，而不是只能缓存一个值
2. 避免使用一个简单的内存对象来做缓存，更好的选择有 lru-cache、redis 以及 memcached 等
3. 缓存函数应该对逻辑函数透明，即逻辑函数是否被缓存，不应该影响整个程序的执行结果

以下给一个实际项目中应用的例子:

```javascript
var q = require('q');
var LRU = require('lru-cache');

function cacheQ(logical, key) {
    var cache = new LRU({
        max: 100, // 最大缓存数量，防止内存泄露
        maxAge: 60000 // 一分钟过期
    });

    return function () {
        var args = arguments;
        key = typeof key === 'function' ? key.apply(this, args) : '__default__';

        // 这里支持 key 作为一个 promise 对象，所以需要用 q 来包装这个对象
        return q(key).then(function (key) {
            if (cache.has(key)) {
                return cache.get(key);
            }

            return logical.apply(this, args).then(function (result) {
                cache.set(key, result);
                return result;
            });
        }.bind(this)); // 注意 this 的传递
    };
}

var readFileCached = cacheQ(function (path) {
    return q.nfcall(fs.readFile, path);
}, function (path) {
    return path;
});
```
