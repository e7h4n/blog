---
layout: post
title: "Mac 下使用代理连接 docker 镜像服务器"
date: 2014-02-26 11:16
comments: true
categories: docker, GFW, proxy, 代理
---

Docker 是个非常牛逼的轻量级虚拟化解决方案，它可以将整个操作系统打成一个非常小的镜像，并且直接跑在宿主的 Linux Kernel 之上。然而在国内因为可恶的 GFW 存在，使用 docker 成了一件非常困难的事情。具体表现在使用 `docker search` 和 `docker pull` 的时候连接非常不稳定，这一刻我又深深的感觉到了作为一名天朝程序员的悲哀...

### 使用 HTTP 代理连接 docker 中央服务器

如果你的宿主操作系统是 linux 那方法就很简单了，直接通过 `HTTP_PROXY=127.0.0.1:8118 HTTPS_PROXY=127.0.0.1:8118 docker -d` 的形式启动服务即可.

### `boot2docker` 使用代理的方法

Mac 下使用 docker 就必须用 `boot2docker` 来创建一个 Virtual Box 虚拟机提供 Linux 宿主环境，要用代理就略微折腾一点。如果只是临时的想用一下，可以通过以下命令来搞定:

```bash
boot2docker ssh # 密码是 tcuser
sudo vi /var/lib/boot2docker/profile
# export HTTP_PROXY=10.0.1.64:8118
# export HTTPS_PROXY=10.0.1.64:8118
/usr/local/etc/init.d/docker restart
```
