title: 使用Hexo搭建Github博客
date: 2015-01-27 19:29:46
tags: [hexo,github]
---
摘要：由于在网上有很多关于创建github博客的教程文章，本文并不复述具体的步骤，而是指出其中的关键步骤，其余的会给出相应参考的链接。

# 前言
虽然Github提供了专门显示静态页面的工具，称为[Github Page](https://help.github.com/categories/github-pages-basics/)。然而要搭建一个成熟的博客，使用Github Page并不太方便。目前搭建github博客的大多数都是使用[Jekyll](http://jekyllrb.com/)、[Octopress](http://octopress.org/)或者[Hexo](http://hexo.io/)。本文主要介绍如何使用Hexo搭建Github博客

# 环境配置
在配置Hexo前需要先配置Node.js、以及Git环境（包括git客户端和Github使用），简明教程可参考文章[http://zipperary.com/2013/05/28/hexo-guide-2/](http://zipperary.com/2013/05/28/hexo-guide-2/)。配置完成后即可开始配置Hexo

## 安装Hexo
利用 npm 命令即可安装。（在任意位置点击鼠标右键，选择Git bash）

``` bash
npm install -g hexo
```

## 初始化博客文件夹
假定你想将博客文件的位置放在E:\blogs处，在命令行到达相应的目录下，执行以下指令：


``` bash
hexo init
npm install
```

## 生成静态页面
由于使用Hexo写文章是使用Markdown语法写在*.md(位于source/_posts/文件夹中)文件里，所以在浏览器上阅读前要先生成相应的静态网页


``` bash
hexo generate   //缩写: hexo g
```

## 本地查看预览
执行如下命令，启动本地服务，进行文章预览调试

``` bash
hexo server   //缩写: hexo s
```

浏览器输入[http://localhost:4000](http://localhost:4000)就可以看到效果

## 写文章
假定要写的文章名为post,那么输入以下命令：

``` bash
hexo new "post"  //缩写: hexo n "post"
```

执行命令后，会在项目source/_posts/中生成post.md文件，用编辑器打开编写即可。当然，也可以直接在source/_posts/中新建一个md文件。

关于Markdown语法可参考以下两篇文章：
[献给写作者的 Markdown 新手指南](http://www.jianshu.com/p/q81RER)
[Markdown 语法说明 (简体中文版)](http://wowubuntu.com/markdown/)

## 部署
编辑_config.yml，并替换相应的repository

	deploy:
		type: github
		repository: https://github.com/njuCZ/njucz.github.io.git
		branch: master

如果是要为一个项目制作网站，那么需要把branch设置为gh-pages
之后执行下列指令即可完成部署:

``` bash
hexo clean
hexo generate
hexo deploy
```

## 主题安装
萝卜白菜各有所爱，玩博客换主题是必不可少的，hexo的主题列表是[Hexo Themes](https://github.com/hexojs/hexo/wiki/Themes),各人可依据自己喜好进行安装。
本博客使用的是pacman的主题，安装方法如下：

``` bash
git clone https://github.com/A-limon/pacman.git themes/pacman
```

安装完成后，打开_config.yml，修改主题为pacman

``` bash
theme:pacman
```

## 博客配置
参考文章[http://zipperary.com/2013/05/29/hexo-guide-3/](http://zipperary.com/2013/05/29/hexo-guide-3/)	

# 回顾

## Hexo命令

常用命令:
- hexo new "postName" #新建文章
- hexo new page "pageName" #新建页面
- hexo generate #生成静态页面至public目录
- hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
- hexo deploy #将.deploy目录部署到GitHub

常用复合命令：
- hexo d -g #生成加部署
- hexo s -g #预览加部署

简写：
- hexo n == hexo new
- hexo g == hexo generate
- hexo s == hexo server
- hexo d == hexo deploy