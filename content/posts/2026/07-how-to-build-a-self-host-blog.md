---
title: 如何搭建一个存在于世界终结的博客
date: 2026-05-16 14:30:00 +0800
tags:
  - Writing
categories:
  - Writing
---

我的博客 `orechou.com`，是我的数字自留地，上面内容的发表、监管不依赖于任何平台，只依赖于现在科技世界的基础设施的存在。也即是说它将存在于这个世界，直至人类文明的消失，如果我不会主动删除它的话。hhh。

而这篇文章就是介绍如何搭建一个这样的 self-host 博客。

## 方案
构建方案涉及到 4 个技术：Hugo、Github Pages、GoDaddy 以及 Cloudflare。

1. Hugo：负责生成网页。用 Markdown 写文章，Hugo 把它变成漂亮的 HTML 网页。它是目前最快的静态网站生成器，建一个站只需要几秒。
2. GitHub Pages：负责托管。把生成的网页文件放到 GitHub 仓库里，GitHub 自动变成一个可以访问的网站。
3. GoDaddy：全球最大的互联网域名注册商和网络托管服务提供商。其实任何一家域名注册商都行，不同的提供商可能有不同的价格优惠，但流程大同小异。
4. Cloudflare：负责 DNS 解析和加速。买了域名之后，需要告诉全世界这个域名指向哪里，Cloudflare 做的就是这件事，同时还能给网站加 CDN 加速和 HTTPS。

## 搭建本地站点

### 安装 Hugo

#### macOS 用户
如果你是苹果电脑，那么安装非常简单，打开终端 app，直接用 brew 安装即可:
```bash
brew install hugo
```
安装完，验证一下：
```bash
hugo version
```
能看到版本号就说明安装好了。

#### 非 macOS 用户
如果是非苹果电脑用户，直接去官网上下载对应的安装包手动安装。[下载链接](https://gohugo.io/installation/)
其它平台我没操作过，大家就自行浏览官方的文档吧。

### 创建博客站点

打开终端 app，进入你想放博客的目录，然后输入命令：
```bash
hugo new site <your-site-name>
cd <your-site-name>
git init
```
这三条命令分别做了：创建一个新的 Hugo 站点、进入站点目录、初始化 Git 仓库。

现在你的站点已经存在了，但它还是个空壳，没有任何主题和内容。

### 安装博客主题

Hugo 官网上已经有很多开发者提交的很多主题了，[地址](https://themes.gohugo.io/)

当然也可以使用我自己开发的一个主题，[hugo-theme-cactus-plus](https://github.com/orechou/hugo-theme-cactus-plus)。因为是自己用，我也在不断地增加新的功能例如：画册、音乐播放等。

安装方法也很简单，直接克隆到你的站点目录即可：
```bash
git clone https://github.com/orechou/hugo-theme-cactus-plus.git themes/cactus-plus
```

后续如果如果我的主题功能有更新，使用命令拉去最新的代码即可：
```bash
cd themes/hugo-theme-cactus-plus
git pull
```

安装完成后，需要在 `config.toml` 中指定主题：
```toml
theme = "hugo-theme-cactus-plus"
```

这样我们就可以看看效果了，继续在终端里面执行：
```bash
hugo server
```
浏览器打开 `http://localhost:1313/` 就可以看到你的博客了。

### 写博客
我们的任何一篇博客，都是在在 `content/posts/` 目录下创建一个新的 Markdown 文件。可以手动创建，也可以用 Hugo 的指令创建：
```bash
hugo new posts/<your-post-name>.md
```

创建的 Markdown 文件的顶部有部分内容叫 Front Matter，它是用来配置博文元数据的。你可以在这个部分填写博文的标题、日期、作者等信息。例如：

```yaml
---
title: "我的第一篇文章"
date: 2026-05-14
draft: true
---
```

## 搭建远程站点

### 创建 Github 仓库
如果没有帐号的，先创建一个 Github 帐号。

创建完 Github 帐号之后，登陆。然后再创建一个仓库。仓库的名称建议用 `<your-username>.github.io` 这样的格式，`<your-username>` 就是你的 Github 用户名。例如我的 Github 用户名是 orechou，那我创建的仓库是 `orechou.github.io`。这样即使我不走后续的流程购买域名，我在互联网上也已经有一个域名可以访问我的博客，即：`https://orechou.github.io`。

### 配置自动部署
在我们本地的站点目录创建一个文件夹 `.github/workflows`，里面创建一个文件 `deploy.yml`，内容如下：

```yaml
name: Deploy to GitHub Pages

on:
  push:
  workflow_dispatch:
  schedule:
    # Runs everyday at 8:00 AM
    - cron: "0 0 * * *"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "latest"

      - name: Build Web
        run: hugo

      - name: Recreate CNAME
        run: echo 'www.orechou.com' > public/CNAME  # 注意：这里使用你的自定义域名

      - name: Deploy Web
        uses: peaceiris/actions-gh-pages@v3
        with:
          personal_token: ${{ secrets.PERSONAL_TOKEN }}
          EXTERNAL_REPOSITORY: OreChou/orechou.github.io # 注意：这里使用你自己的仓库地址
          PUBLISH_BRANCH: master
          PUBLISH_DIR: ./public
          # commit_message: publish blog content
          commit_message: ${{ github.event.head_commit.message }}
```

### 推送你的本地博客到 GitHub
```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/<your-username>/<your-username>.github.io.git
git push -u origin master
```

### 开启 Github Pages
进入仓库页面 → Settings → Pages：
1. 把 Source 设置成 Deploy from a branch 
2. 把 Branch 设置成 master（现在新创建的仓库应该是 main）

然后保存。

![Github Pages Deploy](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/github-pages-deploy.png)

这样我们每次将本地站点内容同步到远程站点之后，GitHub Pages 就会自动构建并部署。

成功后状态会变绿，你的博客就能通过 `https://<your-username>.github.io` 访问了。

### 同步本地站点到远程站点
我们要将本地站点（一个 Git 仓库）同步到远程站点（GitHub Pages 仓库），只需要在本地站点的文件目录下执行：
```bash
git add .
git commit -m "Update"
git push
```
就可以了。这样就会完成站点的内容更新。

## 域名

### 购买域名

在 GoDaddy 注册一个账号之后就可以购买域名了。输入并选择一个心仪的域名即可。
![GoDaddy](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/godaddy.png)
如果有中意的域名购买之后，记得开上自动续费。避免后续因为忘记续费而导致域名过期被其他人注册。

### 配置域名

#### 配置 Github Pages 域名

首先在，进入仓库 → Settings → Pages → Custom domain，填入你的域名（比如 `<your-username>.com`），保存。

![Github Pages Custom Domain](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/github-custom-domain.png)

#### 配置 DNS 

在 Cloudflare 注册一个账号。注册成功后，进入 Cloudflare 控制台，添加你的域名。

然后回到 GoDaddy 的管理页面，把你的域名的 nameserver 改成 Cloudflare 提供的。这里改完生效需要等待全球节点的同步，大概 2 分钟。

![GoDaddy DNS Nameserver](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/godaddy-dns-nameserver.png)

等待 DNS 生效后，回到 Cloudflare 的 DNS 管理页面，添加记录：
添加四条 A 记录，名称填 `@`，分别指向 GitHub Pages 的四个 IP：
- 185.199.108.153
- 185.199.109.153
- 185.199.110.153
- 185.199.111.153

再添加一条 CNAME 记录，名称填 `www`，指向 `username.github.io`。

![Cloudflare DNS Settings](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/cloudflare-dns.png)

#### 配置 HTTPS

在 Cloudflare 的 SSL/TLS 设置里面，在概述页面设置加密模式。我设置的 flexible，适合源站没有证书的情况，仅加密访客到 Cloudflare 的流量。

![Cloudflare HTTPS Flexible](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/cloudflare-flexible.png)

最后再开启始终使用 HTTPS，就做完了最后一步了。
![Cloudflare Enforce HTTPS](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/cloudflare-enforce-https.png)


## 最后
最后，属于你自己的博客就搭建完成了。你讲有两个域名可以访问你的站点：
1. `https://<your-username>.github.io`
2. `https://<your-domain>.com`

如果你不主动删除，它会留存在这个世界直至终结或者 AI 天网统治世界的那一天。

创作愉快。

以上。
