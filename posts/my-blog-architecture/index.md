---
title: "我的博客架构"
date: 2024-01-31T00:00:00+08:00
draft: false
description: "你想只知道我是怎么设计我的系统架构的吗？"
featuredImage: "cover.webp"

tags: ["Blog"]
lightgallery: true
---
<!--more-->

博客人人都写过，很多人都不知道该怎么写博客，让我来教教大家吧（不是）

{{< admonition tip "TL;DR" >}}
使用 `Hugo` 框架，使用 [Doit](https://github.com/cgglyle/DoIt) 主题，使用 `GitHub` 托管源数据， `Vercel` 部署， `waline` 作为评论框架， `Cloudflare` 作为分析和扩展
{{< /admonition >}}

## 简介

我使用如下平台或框架来共同打造这个博客：
 - `GitHub`
 - `IntelliJ IDEA`
 - `Hugo`
 - `Vercel`
 - `waline`
 - [Doit](https://github.com/cgglyle/DoIt)

相信你已经这些框架和平台都很熟悉了，即便不熟悉也没有关系，接下来会详细讲解的。

## 架构

整个博客被我分为 4 个 `Git` 库，包含 `Hugo` 库、主题库、内容库和评论库，这四个库共同构成我的博客。

之所以分为这么复杂的四个步骤，主要考虑到的是隐私的保护，我并不希望我的配置信息和评论区可以被肆意查看。实际上我只希望公开内容库。
现在的情况我的[内容库](https://github.com/cgglyle/blog-content)和[主题库](https://github.com/cgglyle/DoIt)
被公开出来，其他部分都被设为私有。

整个系统由 `Hugo` 库作为主导，内容库和主题库作为 `Git` 子模块被包含。而评论库可以脱离部署。

所有库都被存放在 `GitHub` 中，之后由 `Vercel` 静态部署。（不得不说 `Vercel` 真的好用）之后由 `Cloudflare` 
反代提供保护和分析日志，但是这也会有一个新的问题，国内网络无代理访问速度堪忧，基本上都需要 3 秒以上才能加载。

不过这一套系统因为 `GitHub` `Vercel` `Cloudflare` 的稳定而收益颇多。

## 需要注意的

这里不讲解如何使用 `Git` 子模块，我这里只说一些需要注意的地方

你需要注意 `Vercel` 是不能使用 `SSH` 来拉取仓库的，你的子仓库必须是公开的，并且子模块只
 能使用 `http` 拉取。  

如果你使用 `Doit` 主题的时候你需要注意部署命令，你需要进入 `content` 目录进行部署，而且你需要手动制定各种目录，
这样才可以正常显示 `Hugo` 的 `Gitinfo`

你可以使用这个命令部署：`hugo --cleanDestinationDir --gc --minify --configDir="../config/" --themesDir="../themes" --contentDir="."`

特别的是你需要将 `Root Directory` 设置为 `content`

当你使用这种方式部署时你需要注意各种文件的相对路径，很有可能导致各种问题，需要仔细检查。

## 困扰

隐私也会带来困扰，第一就是繁琐，每次发布博客都很繁琐，除开编写博客，我还需要签名提交 `dev` 版本，`push` 到 `GitHub` 内容库然后 `PR` 
合并到主分支，之后去 `Hugo` 库 `pull` 最新的主分支，签名提交 `dev` 然后等待 `Vercel` 部署预览版，查看是否有问题，
然后 `PR` 合并，等待部署正式版。 更新主题也同理。

这么一套流程要提交两次 `PR` 两次，多少有些繁琐，不过目前也没有什么好的方法处理。
