---
title: 设计一个简洁的 WebApp 预加载策略
date: 2020-04-18 13:21:21
tags:
    - FE
---

这周在跟前端、客户端团队一起设计客户端上 Hybrid 框架的一些技术细节，发现在一些看似简单的方案下，隐藏了非常多细致的工作。

## 问题背景

Hybrid 框架一个非常重要的指标是速度，具体可以细化成秒开率和首屏渲染时间:

* 首屏渲染时间: 从用户点击 WebApp 入口开始，到首屏渲染完成的时间
* 秒开率: 首屏渲染时长小于 200ms 的打开次数 / WebApp 总打开次数

<!-- more -->

用图来表示就是:

从这个指标上很容易看出，尽量缩短首屏渲染时间是主要的解决问题思路。对这个时间的优化，也很容易有以下思路:

* 预下载 (Prefetch): 页面资源预先下载好到客户端，避免 WebView 再去发网络请求
* 预加载 (Preload): 提前在后台打开 WebView

预加载和预下载分别可以节省如下的时间:

从图上可以看出，预加载起到的作用是远远大与预下载的。

## 从最简单的预加载开始

非常直接的思路就是，在客户端启动时，服务器下发一个预加载策略，比如:

```json
{
    "preload": {
        "Home": "https://any-domain.com/any-path?any-param=any-value"
    }
}
```

客户端在切换到首 Tab 时，在后台加载一个不可见的 WebView。当用户点击首 Tab 上某个入口时，客户端判断是否命中预加载的 WebView，如果命中就把预加载的 WebView 展示出来。

## 问题 1: 预加载命中的判断

直接的思路是，如果要加载的页面，和预加载的 WebView 的 URL 完全匹配，就认为是命中。但这个思路，最明显的问题就是限制了预加载在列表页的应用场景。

列表页的场景下，用户有多个出口，每个出口都会有若干个参数来标识下一个页面的资源，例如一个商品列表页:

如果我们用 WebApp 的方案来解决这个问题，每个页面的 URL 都不同。那么该预加载哪些呢？如果要预加载所有商品，内存开销会成为一个很大的问题。如果不是预加载所有，那么这个预加载策略可能会非常的复杂，比如是否在上下滑动的时候要频繁处理 WebView 的新增和销毁？

一个简单的解决方案是用单页面应用 (SPA) 。在这个列表页，所有商品的详情页都应该对应到同一个 SPA 页面上，参数通过某种方式在点击详情页的时候传递进去。这个方案如果命中预加载，可以节省一大部分时间:

TODO

那么，应该如何传递这个参数呢？有以下几种方案:

1. 通过 JsBridge 传递数据
1. `history.pushState` 来改变目标的 URL
1. 修改 URL 的 Anchor 部分

其实无论哪一种方法，目的都是在引起浏览器重定向的情况下，传递数据到 JS 中。我认为最好的方案是修改 URL 的 Anchor 部分。理由如下:

* 修改 Anchor，可以将页面的状态序列化在 URL 上，对刷新友好，同时页面也是可以直接分享的
* 是标准行为，即便目标网页没有逻辑去相应这个变化，这个数据也会保存在 URL 上，不会丢失，更不会引起目标网页的错误

`history.pushState` 也可以实现上面的效果，但会更复杂一些。这会使得客户端需要额外的信息来判断，在什么场景下命中预加载的 WebView。例如，之前服务器下发的预加载指令告诉客户端，在 Home Tab 时需要预加载 `https://any-domain.com/any-path?any-param=any-value`，随后，用户点击了 `https://any-domain.com/any-path/another-path` 这个链接，客户端如何才能知道这种情况下能命中缓存呢？非常困难，这需要额外的约定，例如，约定某个特殊的前缀，或者是用 WebAppId 来标记这两个链接属于同一个 WebApp 下。客户端可能需要预加载、WebApp 本身属性、入口链接与 WebAppId 三套配置:

预加载配置，得用 WebAppKey 而不是 URL:
```json
{
    "preload": {
        "Home": "some-web-app-key"
    }
}
```

WebApp 配置，把 Key 和预加载配置连接起来:
```json
{
    "webAppConfig: {
        "some-web-app-key": {
            "preloadUrl": "https://any-domain.com/any-path?any-param=any-value"
        }
    }
}
```

入口配置，也需要指定 WebAppKey 和目标 URL:
```json
{
...
    "webAppKey": "some-web-app-key",
    "entryUrl": "https://any-domain.com/any-path/another-path"
...
}
```


```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```
