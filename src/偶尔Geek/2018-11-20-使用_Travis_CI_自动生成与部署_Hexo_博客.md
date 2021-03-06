---
layout: post
cid: 213
title: 使用 Travis CI 自动生成与部署 Hexo 博客
slug: 213
date: 2018/11/20 17:08:00
updated: 2019/03/30 17:51:13
status: publish
author: 熊猫小A
categories: 
  - 摸鱼日常
tags: 
excerpt: 不过即使它万般好，评论系统仍然令人头疼。
---


我相信有许多同学选择使用 Hexo + GitHub Pages 作为自己的博客解决方案，例如我的 [Wiki 站点](https://wiki.imalan.cn)即是如此。但是 Hexo 每次写完文章后都要执行一次生成与部署，当文章比较多时这个过程可能变得很慢。

本文介绍了使用 Travis CI 作为自动化手段实现构建、部署 Hexo 博客的过程。这种方法无需手动执行 `hexo g` 与 `hexo d` ，只需在每次编写文章后将更改 push 至远程仓库即可，顺带实现了博客源码备份。使用该方法在第一次配置完成后即使是无 NodeJS 与 Hexo 环境时也可以编写并发布文章，因为构建与部署本身是由 Travis CI 完成的。

## Travis CI

CI 是 Continuous Integration 的缩写，即「持续集成」，本意是帮助开发者在开发过程中随时对小规模的的代码更改进行生成、测试、部署。[Travis CI](https://travis-ci.com/) 提供基于 GitHub 的 CI 服务，它可以从 GitHub 上随时同步代码，并对其进行生成与部署。

或许有同学接触过 GitHub 的 Webhooks 功能，即当代码仓库发生某些操作时，GitHub 会向指定的地址发送请求，当服务端收到这些请求时即可进行下一步操作，例如拉取最新代码-生成-部署。Travis CI 所做的事情与之是类似的。

## 思路

基本的思路如图（诚意手绘）：

![原理流程](./assets/2531426090.jpg)

例如我的代码仓库 [alandecode.github.io](https://github.com/AlanDecode/alandecode.github.io) ，其 master 分支用于存储 Hexo 生成的静态文件，即博客网页；source 分支则用于存储博客源码，一般包括了 `_config.yml`、`source`、`themes`、`package.json` 、`scaffolds` 等文件与目录。

整个工作循环是这样的：在 source 分支下编写新的文章并保存，commit 后 push 至 GitHub 上的 source 分支。Travis CI 探测到 push 操作，即从 source 分支拉取新的源代码，并在自己的服务器上执行生成过程，并将生成结果 push 至代码仓库的 master 分支，完成部署过程。

## 实现

> 假设你已经对 Hexo 的部署与发布有一定的了解，博客已经部署在 xxxx.github.io 这个仓库的 master 分支上。

### GitHub 侧

#### 仓库设置

首先将博客仓库 clone 至本地：

```bash
git clone git@github.com:xxxx/xxxx.github.io.git
```

然后进入代码仓库，创建并切换至新的分支 `source`：

```bash
cd xxxx.github.io
git checkout -b source
```

删除 xxxx.github.io 目录下的所有代码文件（注意不要删除 .git 目录），然后将你的 Hexo 博客源代码尽数拷贝至 xxxx.github.io 目录下。**确保除 xxxx.github.io 根目录下存在 .git 文件夹外，其余位置均不存在**，例如你的第三方主题可能是直接 clone 下来的，里面会有 .git 文件夹，删掉它。

某些临时文件是不必要推送到远端的，为了加快 push 速度，在 xxxx.github.io 下新建 .gitignore 文件，内容为：

```bash
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

检查一下此时 xxxx.github.io 目录下应该包括了 package.json 文件，里面记录了所有的依赖包。此时执行一次 commit 与 push：

```bash
git add .
git commit -m 'init source'
git push --set-upstream origin source
```

此时 source 分支应该已经推送至 GitHub 了。由于以后我们只会修改 source 分支，为了方便将 source 分支设置为默认分支：GitHub 仓库 Settings → Branches → Default Branch 设置为 source。

#### 获取 Access Token

点击 GitHub 右上角头像 → Settings → Developer settings → Personal access tokens，点击 Generate new token，设置 token 名称，并勾选 repo 前面的框。

![access-token.png][2]

点击 Generate，然后将新生成的 token 记下。保管好它，你以后不会再看到这个 token 第二次了。

### Travis CI 侧

去 [Travis CI 官网](https://www.travis-ci.org/)，使用 GitHub 账号登录，然后添加仓库：待仓库列表同步完成后将 xxxx.github.io 勾选。

![add-repo.png][3]

然后点击旁边的 Settings 按钮，如图设置：

![travis-settings.png][4]

下方 Environment Variables 填写 name 为 travis，然后 Value 为 GitHub 获得的 Access Token，取消勾选 Display value in build log，点击 add。

### 添加 travis 配置文件

回到 xxxx.github.io 目录下，创建一个 .travis.yml 的文件，内容为：

```yaml
language: node_js
node_js: stable
install:
  - npm install
script:
  - hexo g
after_script:
  - cd ./public
  - git init
  - git config user.name "【GitHub 用户名】"
  - git config user.email "【GitHub 邮箱】"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${travis}@${GH_REF}" master:master
branches:
  only:
    - source
env:
  global:
    - GH_REF: 【github.com/AlanDecode/alandecode.github.io.git】
```

对应修改其中用【】括起来的部分。再执行一次 commit 与 push：

```bash
git add .
git commit -m 'update'
git push
```

稍等片刻，在 Travis CI 的控制面板将可以看到构建历史：

![build-history.png][5]

若状态为 passed 则说明部署成功。

## 总结

如此下来，Hexo 生成、部署繁琐的问题也没有了。不过即使它万般好，评论系统仍然令人头疼。

以上。

  [2]: ./assets/2355701656.png
  [3]: ./assets/3924164447.png
  [4]: ./assets/3487071668.png
  [5]: ./assets/1849401752.png