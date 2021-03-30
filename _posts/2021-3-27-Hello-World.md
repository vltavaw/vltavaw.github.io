---
title: Hello World:如何用jekyll在github pages上搭建博客
categories:
- General
feature_image: "https://picsum.photos/2560/600?image=872"
---


朋友说在博客的第一篇日志写如何搭博客也是套娃，但我一向无法get到禁止套娃的意义。对我来说，套娃具有递归一样的美感，在适当的情况下是仅次于paradox的优秀手法。更何况记录的一项重要意义就在于此：让自己忘记的时候可以快速找到reference。无论从哪个角度来看，把这一篇作为hw都很合理。~~虽然说这件事也没什么难度就是了~~

#### 如何把博客host到自己的github pages

首先找到[Jekyll Now](https://github.com/barryclark/jekyll-now) 的github页面，并fork。

3.30 edit: 尝试了一下[Alembic](https://jekyllthemes.io/theme/alembic)

打开fork之后的项目，将项目名改为自己的`username.github.io`即可host到自己的github pages。

#### 如何写博客
在根目录下`_config.yml`可以配置博客的相关信息。

![_config.yml]({{ site.baseurl }}/images/config.png)


使用`{{ site.baseurl }}`可以获取到本站的链接。

在`/_posts/`文件夹里管理日志，用md语法即可进行日志的编写。在md的开头加上：
```
---
layout: post
title: Hello World: 如何用jekyll在github pages上搭建博客
---
```
即可完成标题的显示。

更新博客也不一定需要git，可以在任意端完成操作，十分方便。

#### Customization

需要补充更多customization例如tags/不同的view等等。

More later.
