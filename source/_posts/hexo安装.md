---
title: hexo安装
date: 2016-02-29 19:58:22
tags: 
    - 技术
    - markdown
---

折腾了下hexo的环境，用来做自己的markdown博客还是相当不错，还支持github部署。这里讲一下怎么安装。

[hexo官网](https://hexo.io/)
[hexo github地址](https://github.com/hexojs/hexo)

## 安装node.js
hexo是基于nodejs的插件实现，需要先安装nodejs
安装办法参考[nodejs官网](http://nodejs.cn/)。

## 安装hexo
    npm install -g hexo

## 初始化blog
例如我的博客目录在~/blog/，则cd到~目录执行初始化，如下

    cd ~
    hexo init blog
    cd blog
    npm install

## 启动hexo-server
    cd ~/blog/
    hexo server

hexo会启动一个监听4000端口的http服务器，输出信息

    INFO  Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop.

## 生成静态文件
    hexo generate

## 本地查看
在浏览器输入地址`http://localhost:4000/`，即可看到你的博客页面。

## 创建文章
    hexo new "page1"

## github部署
[github page](https://pages.github.com/)是github提供的公开网页托管服务。可以把网页push到github仓库，然后可以通过公开域名访问。

hexo支持进行github page部署。具体步骤如下

1. 在你的github上面创建仓库，名字为'username'.github.io。
    'username'为你的github账号。

2. 配置_config.xml
例如我的github账号为anticpp
```
deploy:
  type: git
  repository: git@github.com:anticpp/anticpp.github.io.git
  branch: master
```


3. 部署
    hexo deploy

`note:`
`如果出现错误信息'ERROR Deployer not found: git'，尝试安装以下组件`


    npm install hexo-deployer-git --save

