title: 使用 gitcafe 做 blog 服务
date: 2014-05-07 15:30:55
tags: hexo
---

由于众所周知的原因，[github] 在国内的访问速度又慢又不稳定，所以[用 github 来托管 blog] 并不是很适合中国国情。所幸国内有 [gitcafe] 这一优质的代码托管服务，可以替代 github 来托管 blog。

使用 gitcafe 非常简单，用过 github 的同学上手肯定毫无压力。创建完 gitcafe 之后新建一个和用户名一样的仓库，例如我的就是 `https://gitcafe.com/perfectworks/perfectworks.git`。

我用的 blog 程序是 [hexo]，部署到 gitcafe 上非常简单，只需要加上几行简单的配置就可以通过 `hexo deploy` 命令来部署:

```yaml _config.yml
deploy:
  type: git
  repository:
    gitcafe: git@gitcafe.com:perfectworks/perfectworks.git,gitcafe-pages
```

注意 `repository` 字段最后要带上 `,gitcafe-pages`，这是 gitcafe 要求的分支名。

如果 blog 之前部署过 github，在用 `hexo deploy` 重新部署前记得先要删除 `.deploy` 目录，否则 `hexo ` 会报错。

部署完成后就可以通过 `http://username.gitcafe.com` 来访问页面了。但是如果 blog 启用了自定义域名，还需要在 gitcafe 中配置一下，以及修改域名的 A 记录。在此不多赘述，gitcafe 的文档写的很清楚: [如何绑定自定义域名信息]。

[github]: http://github.com
[gitcafe]: http://gitcafe.com
[hexo]: http://hexo.io
[如何绑定自定义域名信息]: https://gitcafe.com/GitCafe/Help/wiki/Pages-%E7%9B%B8%E5%85%B3%E5%B8%AE%E5%8A%A9#wiki
