---
title: python-gevent和go-coroutine对比
date: 2016-03-15 10:26:39
tags: 
    - 技术
---

针对python-threading、python-gevent和go-coroutine进行压测对比

## 压测环境
* 压测机器是我的一台阿里云ECS机器
* CPU x86_64 单核
* 内存 1G
* 操作系统Linux iZ94mudtv23Z 2.6.32-431.23.3.el6.x86_64 Centos 6.5
* python 2.7
* go 1.5.1

## 压测模型
* 测试不同并发上下文情况下，进程占用内存、CPU的情况
* 并发数由1000逐步递增到50000

## 压测数据
### 虚拟内存
![bench_vmem](/images/bench_vmem.png)
### 物理内存
![bench_mem](/images/bench_mem.png)
### CPU
![bench_cpu](/images/bench_cpu.png)

## 压测结论
* 总体性能比较go-coroutine > python-gevent > python-threading
* python-threading基本到15000并发数就上不去，可能是系统资源已经到了瓶颈
* python-gevent在并发数较小时表现不错，随着并发数上涨，内存和CPU的增长也较快
* python-gevent一开始就吃虚拟内存比较多
* go-coroutine性能表现出色，随着并发数上涨，内存和CPU占用都维持在比较平稳的水平
* 总的来说不管是python-gevent还是go-coroutine，在用户态进行的context调度，比内核的threading消耗小很多

## 测试代码
[github代码](https://github.com/anticpp/coroutine_benchmark)