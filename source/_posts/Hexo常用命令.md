---
title: Hexo常用命令
date: 2018-03-26 01:24:19
tags: 其它
categories: 其它
keywords: Hexo
description: Hexo常用命令收集
---

## Hexo安装

```shell
> npm install hexo -g #安装
> npm update hexo -g #更新
> hexo init #初始化
```

## 服务器命令

```shell
#启动服务器，Hexo 会监视文件变动并自动更新
hexo server
hexo s

#静态模式
hexo server -s

#修改端口
hexo server -p 5000

#自定义IP
hexo server -i 192.168.1.1

#清除缓存
hexo clean

#生成静态网页
hexo generate
hexo g

#部署
hexo deploy
hexo d

#生成文件并部署
hexo deploy --generate
hexo generate --deploy
hexo d -g
```

## 文章

```shell
#新建文章
hexo new "postName"

#新建页面
hexo new page "pageName"
```