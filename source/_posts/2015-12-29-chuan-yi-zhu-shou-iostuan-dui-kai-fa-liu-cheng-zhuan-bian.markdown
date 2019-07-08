---
layout: post
title: "穿衣助手iOS团队开发流程转变"
date: 2015-12-29 22:18:57 +0800
comments: true
categories: 
---

##原来的开发模式##
团队使用敏捷开发模式

Git Workflow: 使用[Centralized Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/centralized-workflow).

测试: Push Event Hook编译最新版本.

安装: 使用itms-services链接安装APP, 一个静态的html, [iOS测试版本原来的页面](https://ios.ichuanyi.me/)

<!--more-->

###出现的问题###

 - 站会人多, 很多与自己不相关的东西, 浪费时间
 - 发版本时间变长(2星期 -> 一个月)

##调整##
整个开发团队是敏捷开发模式, 但是随着业务增加, 需要多条业务线来保证开发任务可控.
 每条业务线一个小团队(产品/开发/QA), 每个小团队也是采用敏捷开发模式.

##开发##

 - 整体采用[Feature Branch Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow), 一条develop, 多条feature.
 - 每个feature一个小团队(产品/开发/QA)
 - 每个feature branch采用[Centralized Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/centralized-workflow).
 - 使用gitlab来做code review.

##持续集成##

 - QA使用Jenkins编译自己需要的feature branch
 - Jenkins编译完成之后将IPA上传到HTTPS服务器, 根据IPA名字生成一个html, 使用itms-services链接安装APP

##穿衣助手实现方式##
[根据IPA名字生成不同的items-services链接](https://github.com/lvpengwei/iOS-download-manifest)

[穿衣助手测试版本安装(feature branch版)](https://ios.ichuanyi.me/feature/iPhone/html/index.html)


##参考##
[Git - Documentation](https://git-scm.com/doc)

[Git Workflows and Tutorials](https://www.atlassian.com/git/tutorials/comparing-workflows/)