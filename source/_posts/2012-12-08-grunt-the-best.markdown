title: "Grunt -- 最好的前端构建框架"
date: 2012-12-08 11:21
tags: [grunt]
---

每个前端开发工程师都会遇到前端文件打包、压缩的问题。

## Shell -> Ant -> Jake

最开始，我是用 shell 脚本调用 yuicompressor、cssmin 来压缩文件，非常简单，就像这样：

``` sh
#!/usr/bin/env bash

cat src/a.js src/b.js src/c.js > dst/main.js
yuicompressor main.js

cat src/a.css src/b.css src/c.css > dst/main.css
cssmin main.css
```

<!-- more -->

后来随着项目规模的逐渐增大，shell 脚本逐渐暴露出了很多问题，比如：


* 不能自动下载依赖的外部命令，比如 yuicompressor/cssmin
* 缺乏变量替换功能
* 难以跨平台
* 代码难以维护

因此 shell 脚本在到了近百行以后就被放弃了，取而代之的是 Ant。Ant 内置了一些常见任务，比如文件复制、合并、变量替换等，并且可以方便的通过编写自定义任务来扩展。得益于 Java 灵活的扩展性，Ant 任务可以非常方便的在各个项目之间分享，很好的解决了 shell 脚本代码难以维护和迁移的缺点。Ant 还可以结合 ivy 自动解决依赖，实现了完全的跨平台支持。

* 推荐文章：[ant入门指南—web前端开发七武器（1）]

但是，Ant 有个缺点：它是 Java 写的。Ant 如果要执行 js 代码，就必须通过 rhino，速度比 nodejs 慢了不少，作为一个 JavaScript 工程师，我非常希望可以直接用 js 来写构建脚本。

最终让我放弃 Ant 的是一个新的项目。这个项目的特殊之处在于它用了 maven 作为 Java 的构建工具。之前用的各种 Ant 任务也就没有了用武之地。尽管我可以将 Ant 的任务迁移到 mavn 下面，但是我更希望能找到一个和后台语言完全无关的构建系统。

因此在今年年初，我找到了一个 nodejs 环境下的构建框架 [Jake]。Jake 的各种编译任务由 `Jakefile` 中定义（这一点很像 Make），`Jakefile` 实际上就是一个 js 文件，通过 nodejs 来执行压缩合并等常见任务。

得益于 nodejs，Jake 执行 js 代码压缩的速度非常快，而且开发调试也更加方便。但是有个和 shell 脚本类似的问题：不同项目间的 Jake 任务难以重用，项目大了以后 `Jakefile` 依然很难维护，面对一个上千行的 `Jakefile`，和面对一个几百行的 shell 脚本的感觉差不了太多。

以上就是我从 Shell 到 Ant 再到 Jake 的折腾经历。经历这么一圈之后，我发现一个好用的构建框架应该具有以下三个特点：

* 跨平台（Ant, Jake）
* 易维护、易迁移（Ant）
* 开发简单（Shell，Jake）

[grunt] 就是同时具有以上三个特点的前端构建框架。

## grunt

[grunt] 是一个开源的基于任务 (Task) 的前端构建框架。它除了有 Jake 的优点（跨平台、开发简单）以外，还有一套设计良好的 task 框架用来组织各种构建任务。grunt 内置了几个非常常见的构建任务：

* concat - 组合各种文件
* lint - 用 JSHint 检查代码
* min - 用 UglifyJS 压缩代码
* qunit - 跑 QUnit 单元测试
* watch - 当源代码文件发生变化时自动执行任务

除此之外还可以通过 npm 来方便的获取几百个现成的 task，比如用 closure 而不是 UglifyJS 来压缩 js，或者用 less 来生成 css，又或者用 jslint 而不是 jshint 来检查语法等，这些任务都可以在 npm 上找到。如果这些任务无法满足你的需求，grunt 还允许你方便的添加自定任务，就像写 nodejs 代码一样简单。自定任务还可以发布到 npm 上，通过 npm 在多个项目中共享这些任务。[fenbi-grunt-tasks] 就是粉笔网自定的 js 模块合并、handlebars 模板预编译任务 ([grunt-tbf2e] 貌似是淘宝的自定任务)。

任务之间的组合也是 grunt 非常好用的一个特性，例如通过 watch 任务和 rsync 任务的结合，可以方便的实现当源码发生改变时，自动同步代码到服务器上。

每次 grunt 执行时，grunt 都会去读取当前目录下的 `grunt.js` (就像 make 命令去寻找 Makefile 那样)，然后去读取其中的任务配置，例如源码目录等。grunt 最大的特点在于，配置文件中不包含任何的任务逻辑代码(OO 的开闭原则)。这一特性使得任务可以专心于“要做什么”而不是“要对什么做事情”，不再被特定的项目所绑架。

* 推荐阅读：[grunt简要介绍--基于任务的JavaScript项目命令行构建工具]

grunt 给我最大的感受是：原来天下有这么多码农都在为前端构建而奋斗！grunt 使得各个项目的构建脚本不再彼此孤立，使得打造整个公司的前端构建工具变的更加简单。

[Jake]: https://github.com/mde/jake
[grunt]: http://gruntjs.com
[ant入门指南—web前端开发七武器（1）]: http://www.36ria.com/4411
[grunt简要介绍--基于任务的JavaScript项目命令行构建工具]: http://www.oschina.net/question/89964_47198
