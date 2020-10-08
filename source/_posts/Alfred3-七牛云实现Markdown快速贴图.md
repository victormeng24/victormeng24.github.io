---
title: Alfred3+七牛云实现Markdown快速贴图
date: 2019-06-20 10:55:14
tags:
- 工具
---
用Markdown写blog有个比较恶心的问题是，Markdown是纯文本编辑器，不能直接贴图，因此需要一个图床来保存你的图片，在Markdown里面用url的方式引用。
有了图床，每次保存——上传——引用图片url这个步骤又非常繁琐，需要一个简单的workflow来实现一键上传+引用需要的图片，这里推荐的是用Alfred workflow+七牛云图床。
# Alfred3
  需要开通付费版才能自定义你的workflow，关于七牛云图床，我找的是基于qshell(七牛的命令行工具)的workflow，workflow通过调取os+qshell的api，实现将剪贴板或目的路径的图片上传到七牛云，并自动替换剪贴板为该图片url。
![][image-1]
## Workflow
链接:[https://pan.baidu.com/s/1nNfUGMPSaifCYw5okJtK3g][1]  
密码:glzr
## qshell
[https://github.com/qiniu][2]
# 七牛云
自己创建一个对象存储的账户密码即可，免费使用需要实名认证，很简单就不写了～

[1]:	https://pan.baidu.com/s/1nNfUGMPSaifCYw5okJtK3g
[2]:	https://github.com/qiniu

[image-1]:	http://pt1160j8s.bkt.clouddn.com/alfred.png "Alred Workflow"