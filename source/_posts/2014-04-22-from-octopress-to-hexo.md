title: 把 blog 从 octopress 迁移到 hexo
date: 2014-04-22 21:39:27
tags: [octopress, hexo, node]
---

之前用的一直用 [octopress] 来写 blog。对于我来说 octopress 有个最大的缺点，就是它是 ruby 写的。导致我经常就要去搞搞 ruby 的版本啊、gem 啊什么杂七杂八的事情。今天无意间看到了一个 node 写的博客工具 [hex]，功能类似于 octorepss 但是部署成本比 octopress 低了不少，于是花了 15 分钟把整个博客从 octopress 迁移到了 hexo 上面。

和 octopress 类似，hexo 也是一个静态页面生成工具。它允许用 markdown 语法来写文章，并且生成并上传整个网站到 github。它和 octopress 有着完全一样的文章格式，所以整个迁移过程非常轻松。

### 安装 hexo

```bash
# 国内网速慢可以尝试加上 --registry=http://r.cnpmjs.org 来使用国内镜像加速
npm install -g hexo
```
<!-- more -->

### 初始化目录

```bash
hexo init blog && cd blog
```

### 迁移 octopress 的文章

```bash
rm -rf source/_posts
cp -a OCTOPRESS_BLOG/source/_posts source/
```

### 修改链接规则

需要修改 `_config.yml` 中 `new_post_name` 一行:

```yaml
new_post_name: :year-:month-:day-:title.md
```

### 修改博客信息

例如博客名称、网站地址等等，这些内容都在 `_config.yml` 中。另外还要将配置文件最后的 `deploy` 信息修改成 github 地址:

``` yaml
deploy:
  repo: git@github.com:USER/REPO
  type: github
  brandh: master
```

如果博客用了顶级域名，还需要在 `source` 目录下新建一个 `CNAME` 文件:

``` bash
echo "lostjs.com" > source/CNAME
```

### 预览效果

``` bash
hexo generate
hexo server
```

然后浏览器访问本地 4000 端口即可。`hexo server` 和 octopress 的 `rake preview` 命令类似，会自动监听本地文件修改，但是如果修改的是 `_config.yml` 的话就需要重新 `hexo generate` 才能看到效果。

### 发布

``` bash
hexo deploy
```

迁移完毕，浏览器中试一下，原先的 url 都能正常访问。

# 整体感受

hexo 给我的第一感受就是简单，不像 octopress 一样，hexo init 后生成的文件非常少，一眼看过去清除明了。在任何一台机器上只需要 `npm install -g hexo` 之后就能使用，不像 ruby 一样还需要搞 rbenv 并编译 ruby 之类的事情。另外一个感受就是速度快，不管是 generate 还是 deploy 都比 octopress 快了不少。整体来说 hexo 是一个非常值得一试的博客工具。

[octopress]: http://octopress.org
[hexo]: http://hexo.io
