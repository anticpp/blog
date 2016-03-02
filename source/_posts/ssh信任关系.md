---
title: ssh信任关系
date: 2016-02-29 19:26:15
tags: 
    - 技术
---

> 自动ssh登录，我写了一个autossh的工具。
> 有兴趣的朋友可以试试[autossh](https://github.com/anticpp/autossh)

> 回到正题，A要ssh登陆到B，不想输入密码。可以在B建立一个对A的ssh信任关系即可。
> ssh的信任关系是通过rsa实现，具体的操作步骤如下:

## A机器生成rsa key
  - 进入~/.ssh/
  - 运行ssh-keygen，所有的提示都默认回车即可
  - 拷贝公钥，cat id_rsa.pub

## B机器添加公钥
  - 进入~/.ssh/
  - 把A生成的公钥，添加到authorized_keys

## 登陆
  - 从A登陆B机器，无需密码


`注意:`
`非root账号的ssh信任关系一直都还不行，网上很多资料都说是文件权限问题，试了下都不行。`
`所以暂时未解，后面找到原因再跟进到文档。`

