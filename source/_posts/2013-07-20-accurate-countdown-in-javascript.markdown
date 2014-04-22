---
layout: post
title: "setInterval 倒计时陷阱"
date: 2013-07-20 15:03
comments: true
external-url: 
categories: javascript, frontend
---

最近发现页面上的一个倒计时功能出现了比较大的误差，debug 之后发现误差是由于 `setInterval` 不能实现准确执行任务导致的，例如下面这个例子：

```javascript
    var now = +new Date();
    setInterval(function () {
        now += 100;
        console.log('diff', now - (+Date.now()));
    }, 100);
```

<blockquote>
    <div id="demo1">
        <button id="demo1Run">运行</button> <button id="demo1Stop">停止</button>
        <table>
            <tr>
                <td>执行次数：</td>
                <td id="demo1Count">0</td>
            </tr>
            <tr>
                <td>累计误差(ms)：</td>
                <td id="demo1Diff">0</td>
            </tr>
            <tr>
                <td>平均误差(ms)：</td>
                <td id="demo1Avg">0</td>
            </tr>
        </table>
    </div>
</blockquote>

<script type="text/javascript">
(function () {
    var timer = null;
    var countEl = document.getElementById('demo1Count');
    var diffEl = document.getElementById('demo1Diff');
    var avgEl = document.getElementById('demo1Avg');

    document.getElementById('demo1Run').onclick = function () {
        var count = 0;
        var now = +new Date();
        clearInterval(timer);
        timer = setInterval(function () {
            now += 100;
            count += 1;
            countEl.innerHTML = count;
            diffEl.innerHTML = now - (+Date.now());
            avgEl.innerHTML = (now - (+Date.now())) / count;
        }, 100);
    };

    document.getElementById('demo1Stop').onclick = function () {
        clearInterval(timer);
    };
}());
</script>

在我的 chrome 浏览器上平均每次执行时会有 1ms 的延迟。由于这个延迟是累加的，当代码执行 1000 次时误差就积累到了 1 秒。

想到的解决方法就是在执行过程中不停的修正下一次运行函数的时间，由于`setInterval` 的间隔时间是在创建 `timer` 时指定的，所以就只能用 `setTimeout` 来实现定时器。

``` javascript
    var now = +new Date();
    setTimeout(function run() {
        now += 100;
        var fix = now - (+Date.now());
        console.log('diff:', fix)

        setTimeout(run, 100 + fix);
    }, 100);
```

<blockquote>
    <div id="demo2">
        <button id="demo2Run">运行</button> <button id="demo2Stop">停止</button>
        <table>
            <tr>
                <td>执行次数：</td>
                <td id="demo2Count">0</td>
            </tr>
            <tr>
                <td>误差(ms)：</td>
                <td id="demo2Diff">0</td>
            </tr>
            <tr>
                <td>平均每次执行误差(ms)：</td>
                <td id="demo2Avg">0</td>
            </tr>
        </table>
    </div>
</blockquote>

<script type="text/javascript">
(function () {
    var timer = null;
    var countEl = document.getElementById('demo2Count');
    var diffEl = document.getElementById('demo2Diff');
    var avgEl = document.getElementById('demo2Avg');

    document.getElementById('demo2Run').onclick = function () {
        var count = 0;
        var now = +new Date();
        clearInterval(timer);

        timer = setTimeout(function run() {
            now += 1000;

            var diff = now - (+Date.now());

            count += 1;
            countEl.innerHTML = count;
            diffEl.innerHTML = diff;
            avgEl.innerHTML = diff / count;

            timer = setTimeout(run, 1000 + diff);
        }, 1000);
    };

    document.getElementById('demo2Stop').onclick = function () {
        clearInterval(timer);
    };
}());
</script>

用 `setTimeout` 就解决了 `setInterval` 误差累计的问题。

简单封装一下：


```javascript
    function accurateInterval(callback, interval) {
        var now = +new Date();

        setTimeout(function run() {
            now += interval;
            var fix = now - (+Date.now());

            setTimeout(run, interval + fix);

            callback();
        }, interval);
    }
```

末尾，抛几块砖

 * 在 `accurateinterval` 函数中，是否可以先执行 `callback` 函数，再 `setTimeout`？
 * 当 `callback` 函数的执行时间超过定时间隔后会出现什么事情？
 * `accurateinterval` 实现的倒计时，会出现走时不均匀的问题。也就是说倒计时的数字会时快时慢，如何解决这个问题？
