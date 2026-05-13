---
title: 放弃微信读书之后，我搭建了自己的读书系统
date: 2026-05-13 18:00:00 +0800
tags:
  - Book
  - Reading
  - Software
categories:
  - Reading
---

近期，应该说是很久，没有玩游戏了。PS5、Switch 早已吃灰，手机上的 COC 也卸载了。家庭、工作之外的一点自我时间，除了鼓捣一些数码设备之外，其余的都投入到了阅读。这篇文章就分享一下我现在的读书系统。

其实不想折腾的人就用微信读书这一个 app 就好了，体验非常好。但是很遗憾，它现在满足不了我的需求：
1. 有一些冷门书微信读书上是未上架的，一些新书上架也是比较慢的；
2. 一些书畅销书是有删减的，比如《Nexus : A Brief History of Information Networks from the Stone Age to AI》；

这两个原因导致我无法再继续使用微信读书，于是我需要自己找书源、以及管理书籍，更重要的是要有一个比较好阅读体验。

## 系统介绍
我的读书系统由以下部分组成：
1. 书源：z-library
2. 管理：calibre
3. 阅读：Readest + kindle

### 书源
我只有一个书源 z-library，它是一个影子图书馆和开放获取文件分享计划，用户可以在上面下载期刊、文章以及各类书籍。当然，上面的材料基本都是没有版权的，所以我们只用于自身学习，切莫传播。另外一个要点是，z-lib 的假网站非常多，真的网站也因为版权等各种原因被投诉查封。所以我一般是先进入 wiki ，从 wiki 里面找到它最新的地址。
![z-lib](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/z-%20library.png)

### 管理
目前管理书籍的 app，只有 calibre 的功能最强、插件最丰富。
![calibre](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/calibre.png)
导入书籍之后就可以在主页进行管理，还可以对书籍的元书籍进行更新、编辑。元数据的来源可以有豆瓣、Amazon 等可以配置。
![calibre-metadata](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/calibre-metadata.png)
当然 calibre 最大的问题是 UI 不好看，默认图标风格太丑。在设置里面对界面外观以及图标大小进行调整，调整成自己比较喜欢的样式即可。

calibre 的插件很多，如果大家想知道 calibre 的一些更多用法，后续可以再写一篇文章分享

### 阅读

#### Readest 
我是苹果用户，之前是使用 Apple Books 来进行多端的阅读。但后来觉得多端同步需要依赖 iCloud，会占用我本身就少的可怜的 iCloud 空间。以及 Apple Books 的阅读体验我感觉不太好。所以我换成了 Readest 这个 app。它也是支持多平台的同步的，只是免费用户有 500M 的云存储限额，不过对于看文字电子书来说足够用了。

Readest 的主页很简洁，就书籍的预览，以及一个上传按钮。
![readest-homepage](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/readest-homepage.jpeg)

Readest 的阅读界面也很清爽，也有主题可以切换，另外对于多种格式的电子书也渲染的很好。
![readest-reading](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/readest-reading.png)

Readest 是免费的，苹果用户可以直接在 AppStore 里面下载。另外它也是开源的，Github 上可以直接搜到相关源码。

#### Kindle
没错，我的 Kindle Voyage 复活了。10 多年前的产品，当时还是在亚马逊上购买的。随着 Kindle 退出中国，购买电子书不方便后，它放在角落里不知道落灰了多少个年头。现在拿出来看书，阅读体验也还是非常好的。厚度仅为 7.6mm，重量为 180g。轻便、小巧、翻页速度也挺快的。我现在拿它看完了整个《火影忍者》的漫画，找回了读初中时候的乐趣。

calibre 就集成了对 Kindle 书库的管理，用 USB 连上电脑之后，传递书籍非常的方便。
另外 Kindle 阅读最重要的一点是，能够让我足够的专注，不会被手机上各种 App 或者资讯所干扰。这是它最大的价值。
![kindle](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/kindle.jpg)

## One more thing
以上就是我现在主要的阅读系统。当然，阅读的最佳状态，还是一本实体书捧在手上。于是，现在的我也会不断地购买纸质书籍。买那些经典的，值得留下去的书籍。

我愈发觉得在如今 AI 普及、信息密度极高的时代，稳住自己的内心去看书，是非常重要的事情。于此，也是对我的小孩的一种言传身教。我希望她在未来不会丢失看书的能力，没有体会过书籍的乐趣。
