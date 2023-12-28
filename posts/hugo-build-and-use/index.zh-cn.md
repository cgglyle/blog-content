---
title: "Hexo 博客的搭建与使用技巧"
date: 2022-10-30T00:00:00+08:00
draft: false
description: "Hexo 博客的搭建方式和基础使用"
featuredImage: "hexo_fluid_blog_theme_blog.webp"

tags: ["Hexo"]
series: ["Hexo 指北"]
series_weight: 1

lightgallery: true
---

<!--more-->

`Hexo`是一款比较容易搭建的静态博客。可以较为简单的部署到静态网页托管服务器上。甚至不需要花费一毛钱。
{{< admonition tip "Hexo 同类型的静态博客渲染框架" >}}

- `Hugo` 是一款由`Go`语言编写的博客渲染框架，具有很高的渲染速度。
- `vuepress` 是一款由 `Vue` 为编译语言的框架，可以使用 `Vue` 提供的语法。
- 还有很多种，但是只列举出以上两种常用的静态博客框架。

{{</admonition >}}

## Hexo 的搭建
先提前说明一下为什么选择`Hexo`进行搭建。主要是因为 `Hexo`有较多的教程和较为成熟的社区环境。

### 搭建前准备
在搭建 `Hexo` 之前需要做一些准备。(本篇搭建技巧是基于 `Arch Linux` 系统搭建。)
第一点，你要准备好 `Node.js` 这是一款前端开发工程师常会用到的框架。这里只展示如何通过 `Arch Linux` 进行搭建。

 
{{< admonition warning "特别提示" >}}

`sudo` 是系统提权命令，会将权限提升到管理员权限(默认情况下)。请谨慎使用 `sudo` 命令提权，除非你明白那段指令的含义。
但是如果你和我一样正在使用 `Arch Linux` 应该已经对 `sudo` 比较熟悉了。(请允许我介绍我的天主圣父 ArchLinux (笑)）

{{</admonition >}}

```shell
  # 首先下载 Node.js
  sudo pacman -S nodejs
```
之后你需要准备 `Git` 进行版本管理。
```shell
  # 下载 Git
  sudo pacman -S git
```
现在你准备好了所有前置条件。接下来开始正式搭建 `Hexo`
### 正式搭建
首先你需要下载 `Hexo-cli`，这是一个 `Hexo` 的基础集成框架。
```shell
  # 其中 -g 代表全局安装，你也可以选择不全局安装，这取决于你。
  # 如果不使用全局安装可能会出现无法使用命令的情况
  # 这里的 sudo 是因为我的 npm 插件安装目录权限不够所以需要提权命令。
  # 你也可以将目录归属改为你
  sudo npm install -g hexo-cli
```
之后你需要创建一个目录用于存放 `Hexo` 的文件。你可以选择手工创建或者使用命令创建。
```shell
  # mkdir 是创建目录命令，&& 代表这执行完前一个命令之后执行后一个命令，cd 代表打开这个目录。
  mkdir hexo && cd hexo
  ```
现在所有的操作都默认在 hexo 目录下进行操作。你现在需要在目录中进行初始化 `Hexo`
```shell
  hexo init
```
现在你的目录应该呈现以下情况：
```shell
  .
  ├── _config.landscape.yml	# 主题配置文件
  ├── _config.yml				# Hexo 配置文件
  ├── .github					# github 目录
  ├── .gitignore				# git 版本管理忽略文件
  ├── node_modules			# 包目录
  ├── package.json			# 包管理文件
  ├── scaffolds				# Hexo 模板目录
  ├── source					# Hexo 源目录 （我们主要编辑的目录）
  ├── themes					# themes 主题目录
  └── yarn.lock				# 包锁定文件

  5 directories, 5 files
```
到这里，我们的搭建可以说基本完成了。怎么样？是不是很简单。
### 基础使用
在我们完成基础搭建之后，我们需要掌握一些基础的命令，来帮助你使用 `Hexo`
#### hexo s
`hexo s` 这条命令用于通知 `Hexo` 构建一个网页服务出来
```shell
  $ ~/testHexo [1]> hexo s
  INFO  Validating config
  INFO  Start processing
  INFO  Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
```
现在访问 `http://localhost:4000` 就可以看到下面的画面了。

{{< image src="hexo_init_blog_page.webp" caption="初始的 Hexo 页面" >}}

恭喜你，你成功的搭建了一个简易的博客！
接下来我们再介绍几个常用的指令。

#### hexo g
这条命令是用于通知 `Hexo` 进行静态渲染，并在`public`目录下渲染出一个静态的网页。在后续的过程中我们可以吧`public`目录下的文件发送到静态托管服务器进行托管，你就可以使用浏览器搜索的你的博客了。

#### hexo clean
这条命令是用于通知 `Hexo` 对目录进行清理，这会清理掉包括`public`目录在内的所有静态生成目录。推荐在更换主题，或者发现在配置文件中进行配置后，在网页上并没有效果时进行使用。

### 安装主题
可以看到 `Hexo` 的主题页面不是很“新颖”，我们可以选择一个好看的主题进行更换。我们选择使用我正在使用的`hexo-theme-fluid`来装点我们的博客。

```shell
  # 其中 --save 指令是让 npm 将包下载到当前目录，并在 package.json 文件中进行注册。
  npm install hexo-theme-fluid --save
```
当然，你可以直接查看 [fluid 官网](https://github.com/fluid-dev/hexo-theme-fluid) 进行学习安装。
我们需要在项目根目录创建一个`_config.fluid.yml`，并将`node_modules/hexo-theme-fluid/`目录下`_config.yml`文件中的配置完全复制进`_config.fluid.yml`文件中。
```yml
  #---------------------------
  # Hexo Theme Fluid
  # Author: Fluid-dev
  # Github: https://github.com/fluid-dev/hexo-theme-fluid
  #
  # 配置指南: https://hexo.fluid-dev.com/docs/guide/
  # 你可以从指南中获得更详细的说明
  #
  # Guide: https://hexo.fluid-dev.com/docs/en/guide/
  # You can get more detailed help from the guide
  #---------------------------


  #---------------------------
  # 全局
  # Global
  #---------------------------

  # 用于浏览器标签的图标
  # Icon for browser tab
  favicon: /img/fluid.png

  # 用于苹果设备的图标
  # Icon for Apple touch
  apple_touch_icon: /img/fluid.png

  # 浏览器标签页中的标题分隔符，效果： 文章名 - 站点名
  # Title separator in browser tab, eg: article - site
  tab_title_separator: " - "

  # 强制所有链接升级为 HTTPS（适用于图片等资源出现 HTTP 混入报错）
  # Force all links to be HTTPS (applicable to HTTP mixed error)
  force_https: false
```
这里展示部分配置文件。也可以按照 [fluid 官方配置教程](https://hexo.fluid-dev.com/docs/guide/) 进行配置。
我们之后要在项目根目录下的`_config.json`文件中进行配置，以启用主题。在文件的最后可以看到`theme`的配置。我们只需要把它换成`fluid`即可生效。
```json
  // Extensions
  // Plugins: https://hexo.io/plugins/
  // Themes: https://hexo.io/themes/
  // theme: landscape  // after
  theme: fluid // now
```

{{< image src="hexo_fluid_blog_theme_blog.webp" caption="主题初始页面" >}}

现在我们得到了一个看起来很好看的博客。怎么样？是不是很简单。
接下来我们还需要配置一些简单的配置。
```yml
  title: CggLyle's Blog
  subtitle: ''
  description: '无耻的耻骨的博客'
  keywords:
  author: CggLyle
  language: zh-CN
  timezone: ''
```
你可以像我这样进行配置。
### 发布
到现在为止，你已经完成了 70% 的基础配置。你可能在想，那我们应该如何让别人看到我们的博客呢？总不能一直自己看吧？
这是一个好问题，我们有多种多样的方式来解决这个问题。接下来我使用 `GitHub` 方式来实现发布。
#### GitHub 发布方式
`GitHub` 是世界主流的代码托管网站，你可以通俗的理解为你将你的代码存放在 `GitHub` 中，它来帮你保管你的代码，通常情况下，你的代码不会丢失，但是不排除意外情况。
首先你需要申请一个 `GitHub` 帐号，这里会遇到第一个问题，有可能你无法访问 `GitHub` 或者访问相当缓慢。这是因为某种”神秘力量“对你进行了阻碍。你可以选择使用”魔法“来对抗”魔法“，也可以选择更改 `DNS` 进行跨越。但是这不一定有效，这里不做过多讨论。
这里已经默认你成功的创建了一个 `GitHub` 帐号，并进行登录。现在你需要创建一个新的代码库：

{{< image src="github_create_repositories_page.webp" caption="创建仓库" >}}

这里因为我已经创建过一个同名存储库，所以它报红了。你需要创建一个`你的用户名.github.io`为名称的存储库，这是 `GitHub` 提供的一个用于托管网页的专用存储库。
创建之后你会看到一个空的存储库：

{{< image src="github_create_repositories_path_page.webp" caption="仓库地址" >}}

你需要找到你的存储库地址，并将它配置到你项目根目录中`_config.yml`配置文件中。
```yml
  # Deployment
  ## Docs: https://hexo.io/docs/one-command-deployment
  deploy:
    type: git
    repository: git@github.com:cgglyle/cgglyle.github.io.git
    branch: main
```
我这里使用的是 `SSH` 登录方式，这是一种相对安全的登录方式，对于想了解的友人可以等待我其他博文。
接下来我们安装一个新的包，它将提供一个方便的部署方式。
```shell
  npm install hexo-deployer-git --save
```
安装我完成之后就可以使用`hexo d`来将已经静态渲染过的博客推送到 `GitHub` 上。(需要提前使用 `hexo g`进行构建)
我们访问`https://你的用户名.github.io`来查看已经部署好的博客。到此为止，你已经成功的部署 `Hexo`
### 备份
我们现在需要将 `Hexo` 整体进行备份一下。请看这篇博文 [Hexo-博客站点的备份与恢复](../hexo-backup-restore)
## 创建一篇博文
到目前为止，你还没由创建一篇属于你的博文。接下来我就创建一篇新的博文。
### hexo new "我的第一篇博文"
使用`hexo new "我的第一篇博文"`命令可以在`/source/_posts`目录下创建一个名为`我的第一篇博文.md`的 `Markdown`文件。在这里并不介绍 `Markdown`文件的编辑方式。你也可以等待后续可能出现的博文。
你只需要编辑这个文件，撰写你喜欢的博文，然后`hexo clean && hexo g && hexo d`即可完成发布。
**此为止，你已经学会了如何搭建撰写发布一篇博文。**
> 在文章的结尾，我推荐一首我喜欢歌曲吧：[VOODO KINGDOM](https://i.y.qq.com/v8/playsong.html?songid=876829&songtype=0#webchat_redirect)
这同样是`JOJO的奇妙冒险`中人物`DIO`的处刑曲，相当带感。(没有人能在我的BGM中被我战胜！(不是))