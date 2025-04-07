---
title: Hexo+Github Pages+Github Action
date: 2025-04-07 00:00:00
description: 博客部署相关
categories: 
- 折腾
tags:
- Hexo
- Github
---

本博客是使用[Hexo博客框架](https://hexo.io/zh-cn/)+利用Github Pages建立的一个静态站点

唯一的缺点是Hexo编译代码后，基本不能对编译后的文件进行更改，添加新的文章也需要使用Hexo对其重新编译；所以只能在设备部署Hexo, NodeJS之类后才能给新添加的东西进行编译。

解决办法就是利用Github Action之类的进行CICD。这样无需部署任何服务，只要仓库收到了新的提交，即可自动完成上述的一系列操作。（前提是文件、位置啥的符合Hexo规范）

首先了解一下什么是Github Pages和Action吧：
[GitHub Pages 使用入门](https://docs.github.com/zh/pages/getting-started-with-github-pages)
[GitHub Actions 快速入门](https://docs.github.com/zh/actions/writing-workflows/quickstart)

使用也很简单，跟其他CICD区别不大。

步骤：
1. 先给Hexo创建一个本地仓库和远程仓库
2. 在Hexo项目根目录创建`.github/workflows/deploy.yml`文件，并填写工作流
3. 回到Hexo的配置文件`_config.yml`，`deploy: branch`参数改成`gh-pages`
4. 到Github个人设置的开发者设置处添加一个密钥，必须拥有`repo`和`workflow`权限
5. 到Hexo的Github远程仓库`Setting->Security->Actions secrets and variables->action`添加上面生成的密钥，名字写成`GH_TOKEN`
6. 给这个远程仓库创建一个`gh-pages`分支，在远程仓库`Setting->Pages`处，选择刚才创建的分支作为静态页面的文件源
7. 本地推送后，等待action完成即可访问静态页面（`userName.github.io/[repo]`）

>工作流示例：[Github Action's workflow](https://github.com/ECAMT35/ecamt35.github.io/blob/master/.github/workflows/deploy.yml)


