---
layout: post
title: "我的第一篇博客"
date: 2014-03-02 15:24:36 +0800
comments: true
categories:
---

**首先介绍一下blog搭建历程。**

博客管理工具：[octopress](octopress.org)  

当然第一步你得先安装并了解Git. ([Install Git](http://git-scm.com/))

首先要安装**ruby**，当初我是通过rvm来安装，后来出现了一堆麻烦，所以就放弃了这条路，改为HomeBrew->rbenv->ruby。基本不需要配置环境。  

1.[安装Homebrew](http://brew.sh/index_zh-cn.html)
2.[安装rbenv](http://octopress.org/docs/setup/rbenv/)：Alternate Installation Using Homebrew  
3.[安装Ruby](http://octopress.org/docs/setup/rbenv/)：Install Ruby 1.9.3

    ruby --version //查看ruby版本


**安装Octopress**  

    git clone git://github.com/imathis/octopress.git octopress
    cd octopress  
然后安装依赖  

    gem install bundler
    rbenv rehash    # If you use rbenv, rehash to be able to run the bundle command
    bundle install

最后安装Octopress

    rake install

简单配置：主要修改_config.yml，这个配置文件都有相应的注释。主要就是改一些博客头，作者名之类的东西。 注意最好把里面的twitter相关的信息全部删掉，否则由于GFW的原因，将会造成页面load很慢。(from [唐巧](http://blog.devtang.com/blog/2012/02/10/setup-blog-based-on-github/))  


**写博客方法**  

- rake new_post[‘article name’] 生成博文框架，然后修改生成的文件即可
- rake generate 生成静态文件
- rake preview 在本机4000端口生成访问内容
- rake deploy 发布文件

博客内容是采用markdown语法，所以需要熟悉一下常用的标签。我用的是Google Chrome的插件--Minimalist Markdown Editor  

**高级配置还没有测试，下次更新！**


**参考**  

 -  [http://octopress.org/docs/setup/](http://octopress.org/docs/setup/)
 -  [http://octopress.org/docs/setup/](http://octopress.org/docs/setup/)
 -  [破船之家](http://beyondvincent.com/blog/2013/08/03/108-creating-a-github-blog-using-octopress/)
 -  [思考的轨迹](http://shanewfx.github.io/blog/2012/02/16/bulid-blog-by-octopress/)
 -  [唐巧的技术博客](http://blog.devtang.com/blog/2012/02/10/setup-blog-based-on-github/)
