---
title: 使用 Cloudflare Pages 部署网站
date: 2021-11-14 21:04:34
tags:
---

使用 Cloudflare Pages 从 GitHub 帐户直接部署前端应用。
在开始使用 Cloudflare Pages，并将网站部署到 Pages 平台前需要先创建一个 [Cloudflare 账户](https://dash.cloudflare.com/sign-up)

## 目录

- 连接 GitHub 帐户
- 配置和部署

### 连接 GitHub 帐户

#### 登录 Cloudflare Pages
首先，打开[Cloudflare Pages](https://pages.cloudflare.com/) 站点管理并使用 Cloudflare 帐户登录。如果还没有帐户，需要先进行注册。

#### GitHub授权
在`Cloudflare Pages`页面中选择`创建项目`并将`GitHub`账户添加进来。添加完成后在页面中可以看到`GitHub`的存储库
![1](http://jsnext.icu/20211114212219.png)

### 配置和部署

#### 选择 GitHub 存储库
可以从个人帐户或授予 `Pages` 访问权限的组织中选择一个 `GitHub` 项目。允许选择一个 `GitHub` 存储库使用 `Pages` 进行部署。支持私有和公共存储库。

#### 基础配置
选择存储库后，选择`Install & Authorize and Begin setup`。然后，可以在设置构建和部署页面中自定义部署参数。

项目名称：用于生成访问页面的域名，默认值与`GitHub`项目名称相同

分支：Cloudflare Pages 应用于部署站点的版本分支
![2](http://jsnext.icu/20211114213047.png)

#### 构建配置
根据需要部署到 Cloudflare Pages 项目的 `框架`，需要指定站点的构建`命令`和`输出目录`，从而告诉 Cloudflare Pages 如何部署站点。输出目录的内容作为需要上传到 Cloudflare Pages。配置完成后点击`保存并部署`。
`根据项目框架的不同可能有些环境需要有单独的配置，例如构建ng13的项目所依赖的node版本必须14+以上`
![3](http://jsnext.icu/20211114213901.png)

在部署的过程中，可以看到控制台的一些输出(这里有可能会报错，根据控制台的error信息自行处理即可)。编译和部署完后可以看到以下结果。
![4](http://jsnext.icu/20211114214001.png)

在回到pages页面可以看到已经部署好的网站
![5](http://jsnext.icu/20211114214448.png)