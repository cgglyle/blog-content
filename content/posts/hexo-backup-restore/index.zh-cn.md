---
title: "Hexo 博客站点的备份与恢复"
date: 2022-10-21T00:00:00+08:00
draft: false
description: "相信你也不想你站点炸掉的事情被读者知道吧。（笑"
featuredImage: "featured-image.webp"

tags: ["Hexo"]

lightgallery: true
---

<!--more-->

费劲建好的博客站点肯定是不希望因为一些奇怪的原因导致站点崩溃，导致最后丢失数据。

## 备份
对于我来说备份的选择就是直接使用 Git 进行备份，就很方便。只需要排除一些文件就好了。

### 需要保留的文件
先来看一下正常情况下一个 Hexo 博客站点包含以下文件目录。

```bash
    .
    ├── _config.fluid.yml   # 主题配置文件
    ├── _config.yml         # Hexo 配置文件
    ├── db.json             # 不明，但是不重要，是自动生成的
    ├── .deploy_git         # 最终由插件 hexo-deploy-git 上传的文件
    ├── .gitignore          # git 排除文件
    ├── node_modules        # npm 包文件
    ├── package.json        # 包配置文件
    ├── package-lock.json   # 包锁定文件
    ├── public              # Hexo 生成的静态网页
    ├── scaffolds           # 模板文件
    ├── source              # 原始文章目录
    ├── themes              # 主题目录
    └── yarn.lock           # yarn 锁定文件
```

其中 `public` 是执行 `hexo g` 静态生成的，所以可以直接无视，不用备份。`source` 是存放文章原始记录的位置，需要备份。

`themes` 是存放主题文件的位置，如果你使用 `git` 来下载的主题文件，这里建议不用备份这个目录，在需要恢复的时候直接使用 git 来重新下载就好。

其中 `scaffolds` 目录是存放模板文件的位置。如果有大量模板建议 git 上传。

#### 正式备份的文件

```bash
    ├── _config.fluid.yml   # 主题配置文件
    ├── _config.yml         # Hexo 配置文件
    ├── package.json        # 包配置文件
    ├── scaffolds           # 模板文件
    ├── source              # 原始文章目录
    ├── .gitignore          # git 排除文件
    └── themes              # 主题目录
```

我们直接在 GitHub 上开一个仓库，然后将这写文件上传上去就好了。下面附上一个 `.gitigonre` 文件的内容。

```.gitigonre
    .DS_Store
    Thumbs.db
    db.json
    *.log
    node_modules/
    public/
    .deploy*/
    _multiconfig.yml
    package-lock.json
    yarn.lock
    .github
```

## 恢复
假设你的 Hexo 出现了问题，需要你恢复备份。

### 操作

首先需要将你的 Hexo 备份从存储库中拉下来，之后进入 Hexo 目录执行 `npm install` 就可以自动处理依赖。

如果你没有安装 Hexo-cli 需要安装以下，并设置好的主题目录，之后就可以正常执行了。