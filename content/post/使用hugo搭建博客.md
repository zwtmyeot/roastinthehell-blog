---
title: "使用hugo搭建博客"
date: 2020-03-20T08:52:24+08:00
draft: true
tags: [
    "hugo"
    ]
---

记录一下使用hugo搭建本博客的过程<!--more-->

## 安装Hugo

使用brew安装hugo：

```sh
brew install hugo
```

验证是否安装完成

```sh
hugo version
```

## 新建站点

```sh
hugo new site MySitev
```

## 下载主题

选择一个自己喜欢的主题：[Hugo Themes](https://themes.gohugo.io/)

这里使用git子模块的方式下载：

```sh
cd Mysite/
# 初始化git仓库
git init
# 下载主题到themes文件夹
vgit submodule add https://github.com/matsuyoshi30/harbor.git themes/harbor
```

在配置文件 `config.toml` 中选择下载的主题，同时进行一些配置：

```conf
baseURL = ""
title = "My New Hugo Site"
theme = "harbor"
```

## 新建一个文章

```sh
hugo new post/my-first-post.md
```

### 主题与CSS渲染的关系

我使用的是 [harbor](https://github.com/matsuyoshi30/harbor) 这个主题，它的一些CSS式样是和 `hugo new post/my-first-post.md` 中生成的路径 `post` 有关系，如果使用 [官方教程](https://gohugo.io/getting-started/quick-start/#step-4-add-some-content) 中的命令：

```sh
hugo new posts/my-first-post.md
```

那么会生成一个 `posts` 文件夹存放文章，这样使用hugo渲染后文章会缺失一些CSS式样

## 开启Hugo 服务器

```sh
# -D 是把draft文件，即草稿文件也编译预览
hugo server -D
```

打开 [http://localhost:1313/](https://localhost:1313/) 进行预览

## 编译文章

```sh
# -D 是把draft文件也编译了
hugo -D
```
