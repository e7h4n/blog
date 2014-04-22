---
layout: post
title: "教程：使用 pathogen + git 管理 Vim 插件"
date: 2012-02-04 02:40
comments: true
categories: [Vim, Git]
---

Vim 用了很多年，一开始我是把所有插件直接扔到 .vim 目录下。时间长了以后，.vim 下面有好多零碎的文件，分不清哪个属于哪个插件，删除和升级插件都很困难。相信很多 Vimer 也遇到了这个问题。

Pathogen 是一款拯救各个 Vimer 于水火之中的 Vim 插件，它一改原先 Vim 只能把插件全部扔到 .vim 目录下的操作方式，使得各个插件可以以一个独立的文件夹存在于 .vim/bundle 目录中，添加和删除插件都变的非常清爽。再加上 git 强大的子模块管理功能，可以实现方便的插件安装和自动升级。

<!-- more -->
## Pathogen 做了什么

在 Pathogen 之前，安装插件就是把文件都丢到 .vim 目录下，文件都混在一起，非常不好管理：

    .vim
      ├── doc
      ├── plugin
      │   ├── vim-scratch.vim
      │   └── vim-surround.vim
      ├── ftplugin
      └── autoload

Pathogen 可以理解成一个插件的加载器，通过 Pathogen，可以将不同的插件放到不同的目录里，比如：

    .vim
      └──bundle
          ├── vim-scratch
          │   └── plugin
          │       └── vim-scratch.vim
          └── vim-surround
              ├── doc
              └── plugin
                  └──vim-surround.vim

这样，各个插件之间的文件都独立于自己的目录，删除一个插件，只要直接删除这个插件的目录就行了。

Pathogen 的使用非常简单，在此不在赘述，参见文章 [更好的管理VIM插件（续） pathogen](http://blog.syndim.org/2011/08/13/vim-pathogen/)

## Pathogen 做不了什么

Pathogen 相当于一个插件的加载器，只提供最简单的插件加载功能，没有提供任何插件的安装、删除、更新这些管理功能。估计作者也是本着 Unix 哲学（程序应该只关注一个目标，并尽可能把它做好）思想，将来也不会加入任何管理功能。

但是如果要打造一套完善的 Vim 插件方案，一个靠谱的插件自动化管理系统是必不可少的。就像 Chrome 的插件系统一样，插件可以：

* 自动安装，新拿到一台电脑，可以方便地自动下载插件
* 自动化升级，不需要繁琐的文件拷贝更新操作

因此就有了 Vundle 这类工具，通过一系列的脚本，来实现这些功能。

## Pathogen + Git

我自己比较喜欢的方案是 Git + Pathogen，本质上和 Vundle 没有区别（Git 就是各种牛逼的脚本组合起来的），但是有着很多让我着迷的优点：

* 只依赖于 Git（Pathogen 通过 Git 下载）
* Github 上几乎有全部的 vim 插件库
* 符合 Unix 哲学，Pathogen 做加载，Git 做插件管理，这个组合非常漂亮

### 准备工作

首先，备份你原先的 .vim 配置，并创建一个新的 .vim 目录，以及放置插件的 bundle 目录：

``` bash
$ mv .vim{,.bak}
$ mv .vimrc{,.bak}
$ mkdir -pv .vim/bundle
> .vim
> .vim/bundle
```

然后把 .vim 目录变成一个 Git 仓库。做这一步非常简单，切换到 .vim 目录下，执行``git init``命令，git 会创建一个 .git 目录：

``` bash
$ cd .vim && git init
> Initialized empty Git repository in /Users/pw/.vim/.git/
$ ls -al
> total 0
> drwxr-xr-x   4 pw  staff  136 Feb  4 14:01 .
> drwxr-xr-x   4 pw  staff  136 Feb  4 14:01 ..
> drwxr-xr-x  10 pw  staff  340 Feb  4 14:01 .git
> drwxr-xr-x   2 pw  staff   68 Feb  4 14:01 bundle
```

至此，准备工作就完成。以下的命令如果没有特别说明，都是在 .vim 这个目录中敲入的。

### 安装 Pathogen

安装插件的命令是：

``` bash
$ git submodule add 插件的Git仓库地址 bundle/插件名字
```

Pathogen 将会是我们通过 Git 安装的第一个插件：

``` bash
$ git submodule add git://github.com/tpope/vim-pathogen.git bundle/vim-pathogen
> Cloning into bundle/pathogen...
> remote: Counting objects: 218, done.
> remote: Compressing objects: 100% (117/117), done.
> remote: Total 218 (delta 59), reused 202 (delta 45)
> Receiving objects: 100% (218/218), 26.40 KiB | 23 KiB/s, done.
> Resolving deltas: 100% (59/59), done.
```

通常来讲，一个插件下载完之后就已经可以使用了，但是对于 Pathogen 这个”插件中的插件“来说，还要多一，创建一个非常简单的 .vimrc 文件，加载 pathogen

``` bash
$ echo -e "runtime bundle/vim-pathogen/autoload/pathogen.vim\ncall pathogen#infect()\nHelptags" >> .vimrc
$ ln -sf `pwd`/.vimrc $HOME/
```

### 安装其他插件

方法跟安装 Pathogen 是一样的，在 .vim 目录下执行：

``` bash
$ git submodule add 插件的Git仓库地址 bundle/插件名字
```

以 NERDTree 为例，仓库地址是``git://github.com/scrooloose/nerdtree.git``：

``` bash
$ git submodule add git://github.com/scrooloose/nerdtree.git bundle/nerdtree
```

### 升级插件

单独升级插件，只要先进入插件目录，然后执行：

``` bash
git checkout master; git pull
```

通过``git submodule foreach``来可以一次性升级全部插件：

``` bash
$ git submodule foreach 'git checkout master && git pull'
```

### 删除插件

删除一个插件稍微繁琐了一点（相比较添加和升级），需要两条命令：

``` bash
$ rm -rf bundle/插件名
$ git rm -r bundle/插件名
```

### 把整个 .vim 目录发布到 Github 上去

如果你是一直按照我的教程做到这里，这时你可以通过``git status``查看一下 .vim 这个 Git 仓库的状态：

``` bash
$ git status
> # On branch master
> #
> # Initial commit
> #
> # Changes to be committed:
> #   (use "git rm --cached <file>..." to unstage)
> #
> #   new file:   .gitmodules
> #   new file:   bundle/nerdtree
> #   new file:   bundle/pathogen
> #
> # Untracked files:
> #   (use "git add <file>..." to include in what will be committed)
> #
> #   .vimrc
```

可以看到 Git 的暂存区中还没有加入 .vimrc（如果你不懂什么是 Git 暂存区，没关系，按照教程继续往下走，不过最好去看一下 Pro Git 这本书），将 .vimrc 加入暂存，并提交：

``` bash
$ git add .vimrc
$ git commit -m 'ADD: pathogen & nerdtree'
> master (root-commit) ba3afbd] ADD: pathogen & nerdtree
> 4 files changed, 11 insertions(+), 0 deletions(-)
>  create mode 100644 .gitmodules
>  create mode 100644 .vimrc
>  create mode 160000 bundle/nerdtree
>  create mode 160000 bundle/pathogen
```

在做下一步之前，你需要在 github 上开一个远程仓库（这一步就不给出具体教程了，非常简单，在 Github 上操作）。然后把本地 Git 仓库推送到 Github 上。例如，我的 vim 仓库地址是``git@github.com:perfectworks/vim.git``：

``` bash
$ git remote add origin git@github.com:perfectworks/vim.git
$ git push origin master
```

然后去 Github 上看看，可以发现，所有通过 Git 加入的 Vim 插件都是以一个链接直接链到插件的 Github 仓库，看起来非常整齐漂亮。
