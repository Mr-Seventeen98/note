---
title: npm私仓搭建
date: 2022-04-17 13:48:57
tags:
---

在一个中大型企业中，可以将私有包托管在私仓上，不对外发布。在项目中使用的包可以缓存在私仓上，down下来的速度明显要快。下载、发布npm包都有自己的权限管理。在团队中，可以激励自己与其他成员的开源、创新能力，共同提高。

## 目录
在这篇文章中，我们将讨论以下主题：
- 使用 Verdaccio 搭建npm私仓
- 使用 pm2 管理 Verdaccio 进程
- 发布 npm 包

## 环境依赖
- node
- npm

### 使用 Verdaccio 搭建npm私仓
服务器上全局安装 Verdaccio
``` bash
npm install -g Verdaccio
```
修改 `config file` 配置文件，在配置文件最后添加监听端口,保存并退出。
``` bash
listen: 0.0.0.0:4873
```
确保 `liunx` 对外开放了 `4873` 端口（--zone #作用域 --permanent #永久生效）
``` bash
firewall-cmd --zone=public --add-port=4873/tcp --permanent
```
重新载入
``` bash
firewall-cmd --reload
```
查看是否添加成功,执行以下命令，返回 `yes` 说明端口已开放
``` bash
firewall-cmd --zone=public --query-port=4873/tcp
```
启动verdaccio
``` bash
verdaccio
```
启动成功后，访问服务器外网，host:4873，此时会看到
![](http://jsnext.icu/note/npm/npm01.jpg)

但这样启动的是前台进程，当 `liunx` 需要做其他操作的时候，当前进程就会停掉，导致不能访问。所以我们需要借助 `npm2` 来管理 `Verdaccio` 进程。

### 使用 pm2 管理 Verdaccio 进程
pm2守护进程 如果未安装pm2，先通过npm安装：
``` bash
npm install pm2 -g
```
接下来，用pm2来启动verdaccio (-i 4 让 verdaccio 进程占满整台服务器的资源)
``` bash
npm2 start verdaccio -i 4
```
### 发布 npm 包
发布
``` bash
npm publish --registry http://192.168.31.149:4873
```
> 当已经发布完成后，之后更新就是用`npm version patch`更新版本号，之后再次执行发布命令



