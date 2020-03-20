---
title: "使用Hugo搭建博客并自动部署到VPS"
date: 2020-03-20T08:52:24+08:00
draft: false
tags: [
    "hugo"
    ]
---

记录一下使用hugo搭建本博客的过程<!--more-->

使用Hugo生成静态博客网站，然后推送到GitHub仓库，通过GitHub的webhook通知vps上的caddy，然后caddy拉取项目，再渲染生成新的文章，达到自动部署的目的

## 安装Hugo

### 在Mac上安装

使用brew安装hugo：

```sh
brew install hugo
```

### 在Linux上安装

**不要使用包管理器安装！**

我在Debian上使用apt安装hugo是有问题的，版本特别落后，会导致编译文章失败

请安装二进制文件（可执行程序）

先从 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 页面找到自己的系统版本，我使用的是Debian，选择 [hugo_0.67.1_Linux-64bit.tar.gz](hugo_0.67.1_Linux-64bit.tar.gz)

```sh
# 新建一个文件放下载文件
mkdir hugo
cd hugo
# 下载压缩文件
wget https://github.com/gohugoio/hugo/releases/download/v0.67.1/hugo_0.67.1_Linux-64bit.tar.gz
# 解压文件
tar xvf hugo_0.67.1_Linux-64bit.tar.gz
# 把hugo的可执行文件添加到shell路径中
sudo mv hugo /usr/local/bin
```

验证是否安装完成

```sh
hugo version
```

## 新建站点

```sh
hugo new site MySite
```

新建好博客站点后，初始化git仓库：

```sh
cd MySite

git init
```

## 使用主题

选择一个自己喜欢的主题：[Hugo Themes](https://themes.gohugo.io/)

这里使用git子模块的方式下载：

```sh
cd Mysite/
# 初始化git仓库
git init
# 下载主题到themes文件夹
git submodule add https://github.com/matsuyoshi30/harbor.git themes/harbor
```

在配置文件 `config.toml` 中选择下载的主题，同时添加自己的信息：

```conf
baseURL = "https://blog.roastinthehell.com/"
languageCode = "zh-cn"
title = "Roast in the Hell"
theme = "harbor"

paginate = 5
DefaultContentLanguage = "zh-cn"
enableInlineShortcodes = true
footnoteReturnLinkContents = "^"

[Author]
  name = "wayZ"

[outputs]
    section = ["JSON", "HTML"]

[[params.nav]]
  identifier = "about"
  name = "About"
  icon = "fas fa-user fa-lg"
  url = "/about/"
  weight = 3

[[params.nav]]
  identifier = "tags"
  name = "Tags"
  icon = "fas fa-tag fa-lg"
  url = "tags"
  weight = 3

[params]
  mainSections = ["post"]
  favicon = "favicon.ico"
  custom_css = ["css/custom_style.css"]

  [params.logo]
    url = "icon.jpg"
    width = 50
    height = 50
    alt = "Logo"
```

`baseURL` 是渲染后页面中资源的路径，不要填错了，不然会缺失 CSS 、 JavaScript 脚本

## 新建一个文章

```sh
hugo new post/my-first-post.md
```

我使用的是 [harbor](https://github.com/matsuyoshi30/harbor) 这个主题，它的一些CSS式样是和 `hugo new post/my-first-post.md` 中生成的路径 `post` 有关系，如果使用 [官方教程](https://gohugo.io/getting-started/quick-start/#step-4-add-some-content) 中的命令：

```sh
hugo new posts/my-first-post.md
```

那么会生成一个 `posts` 文件夹存放文章，这样使用hugo渲染后文章会缺失一些CSS式样

所以如果你发现自己文章渲染后的主题式样有问题，请检查一下主题模板中是否有些式样和路径绑定了

## 开启Hugo 服务器

```sh
# -D 是把draft文件，即草稿文件也编译预览
hugo server -D
```

打开 [http://localhost:1313/](https://localhost:1313/) 进行预览

可以看见已经生成了文章

## 编译文章

```sh
# -D 是把draft文件也编译了
hugo -D
```

在正式发布的时候，请手动把文章开头的草稿标记 `draft: true` 改为 `draft: false`

## 部署

现在本地博客站点已经搭建完成，下一步就是考虑部署

我有自己的 VPS ，所以我打算放在自己的 VPS 上

如果你没有自己的 VPS ，可以考虑放在GitHub Page

在开始前先把自己博客域名的解析先添加了，毕竟要一点时间才能生效

## 设置GitHub webhook

在GitHub中新建一个仓库，把博客文件夹上传到GitHub中，然后在博客项目中设置webhook

在博客仓库中选择：`Settings--Webhooks--Add webhook`

- `Payload Url` 选项填写`自己博客地址/webhook`，例如我的是： `https://blog.roastinthehell.com/webhook`

- `Content type` 选择 `application/json`

- `Secret` 填一个自己的密码，用于通信时使用

## 安装caddy

打开 [caddy官网](https://caddyserver.com/v1/download)，选择使用脚本安装的方式

系统按照你自己的vps选择，我选择了 Linux 64-bit 版本

`plugins` 中添加 `http.git` 模块

拉到下面可以看到一键脚本安装链接：

```sh
# 最后的http.git就是刚才选择后添加上的
curl https://getcaddy.com | bash -s personal http.git
```

会把caddy放到：`/usr/local/bin`

## 配置开机启动

使用systemd服务来实现后台运行及开机启动：[systemd Service Unit for Caddy](https://github.com/mholt/caddy/tree/master/dist/init/linux-systemd)

配置好以后可以通过 `systemctl` 命令查看状态、停止或者重启caddy：

```sh
# 查看caddy的状态
systemctl status caddy.service
# 开启caddy
systemctl start caddy.service
# 重启caddy
systemctl restart caddy.service
# 停止caddy
systemctl stop caddy.service
```

### Caddyfile配置文件

通过 `systemctl` 启动的caddy会默认读取 `/etc/caddy/Caddyfile` 作为配置文件

新建然后修改配置文件：

```sh
sudo touch /etc/caddy/Caddyfile
sudo vim /etc/caddy/Caddyfile
```

填入一下内容：

```conf
blog.roastinthehell.com {
    # 启用gzip
    gzip
    # 启用https，需要预先在dns解析中添加域名解析，caddy会自动帮你申请证书
    tls zwsghxs2@gmail.com
    # 网站主目录
    root /var/www/blog/
    # log文件，记录访问记录
    log /var/www/logs/blog.log
    header / {
    # 配置 HSTS（HTTP严格安全传输）
    Strict-Transport-Security "max-age=31536000;includeSubdomains;preload"
    }

    # webhook相应的设置
    git {
        # 你的GitHub仓库地址
        # 这里使用https的方式克隆
        repo https://github.com/zwtmyeot/roastinthehell-blog.git
        # 如果你的仓库配置了ssh密钥推送，使用ssh克隆，必须要有私钥才能克隆
        # key /home/adam/.ssh/id_rsa
        branch master
        # 克隆到哪里，这里是放在了roastinthehell-blog文件夹
        path /var/www/roastinthehell-blog
        # 克隆时同时拉取子模块，这里是把我们的hugo主题也拉取下来
        clone_args --recursive
        # 拉取博客项目是，如果子模块有更新也拉取更新
        pull_args --recurse-submodules
        # then后面的就是收到GitHub webhook通知后执行的命令
        # 编译后把生成的文件放在网站主目录
        then hugo --source=/var/www/roastinthehell-blog/  --destination=/var/www/blog
        hook /webhook 填GitHub中设置的通信密码
        hook_type github
    }
}
```

编辑好caddy的配置文件以后，重启caddy服务：

```sh
sudo systemctl restart caddy.service

# 然后查看caddy服务的状态
systemctl status caddy.service
```

如果caddy服务启动失败，那可能是 caddy中的git配置部分出问题，有可能是拉取有问题，也有可能是hugo编译文章有问题

使用 `journalctl` 工具查看caddy的记录：

```sh
journalctl --boot -u caddy.service
```

输入 `G` 可以滚动到最后，一条一条查看，看是哪里出问题

如果都没有问题，修改一下文章，推送到GitHub，刷新一下博客主页就可以看到变化了

祝愿大家都能有持续旺盛的精力，不断地写博客 ;-)

## 参考

- [利用Caddy实现Hugo个人博客的自动化部署](https://blog.wangjunfeng.com/post/caddy-hugo/)

- [通过github和caddy实现hugo的自动部署](https://itlaws.cn/post/hugo-caddy-autodeplay/link)