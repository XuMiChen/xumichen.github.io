---
title: 为什么nat模式不配置widnows防火墙的时候wsl可以访问win服务比如python3，但是不能ping通
date: 2025-04-23 10:00:00
tags:
  - wsl
  - 网络
  - Windows
  - python
categories:
  - 技术
---


## win下启动一个 python server
![](../img/posts/wsl-windows-python/1.png)

## 发现wsl可以访问
![](../img/posts/wsl-windows-python/2.png)

## 原因是python启动服务的时候会弹出防火墙规则，选择确认之后会创建两个规则，tcp和udp
![](../img/posts/wsl-windows-python/3.png)

## 验证: 把tcp规则关闭
![](../img/posts/wsl-windows-python/4.png)
![](../img/posts/wsl-windows-python/5.png)

## 此时不能访问
![](../img/posts/wsl-windows-python/6.png)

## ping不通的原因
> ping命令不属于python的防火墙规则，windows默认的规则是不允许被ping

### 参考
> Ping命令的工作过程及单向Ping通的原因
https://www.cnblogs.com/yxmx/articles/1547375.html
