---
title: keepalived入门
date: '2019/3/26 10:23:41'
categories:
  - Web
tags:
  - 分布式
abbrlink: 3221
---
# keepalived入门

## keepalived简介
>&emsp;&emsp;keepalived是集群管理中保证集群高可用的一个服务软件，用来防止单点故障。

>&emsp;&emsp;keepalived的作用是检测web服务器的状态，如果有一台web服务器死机，
或工作出现故障，keepalived将检测到，并将有故障的web服务器从系统中剔除，当web服务器
工作正常后，keepalived自动将web服务器加入到服务器集群中，这些工作全部自动完成，不需要
人工干涉，需要人工做的只是修复故障的web服务器。

>&emsp;&emsp;keepalived是以VRRP协议位实现基础的，VRRP全称 Virtual Router
Redundancy Protocol，即虚拟路由冗余协议。

>&emsp;&emsp;虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同
功能的路由器组成的一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务
的VIP（Virtual IPAddress,虚拟IP地址，该路由器所在局域网内其他机器的默认路由为该VIP），
master会发组播，当backup收不到VRRP包时，就认为master宕掉了，这时就需要根据VRRP的优先级来
就可以保证路由器的高可用了。

<!--more-->

### keepalived核心内容

>&emsp;&emsp;keepalived主要有三个模块，分别是core、check和VRRP。core模块为
keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查
，包括常见的各种检查。VRRP模块是来实现VRRP协议的。

初始状态：
>
![初始状态](/images/keep1.bmp)

主机状态：
>
![主机状态](/images/keep2.bmp)

主机恢复：
>
![主机恢复](/images/keep3.bmp)

