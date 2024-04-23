---
title: PVE8.0下为LCX添加TUN接口
categories:
  - 虚拟化
tags:
  - PVE
  - LCX
  - TUN
  - ZeroTier
---

# PVE8.0下为LCX添加TUN接口

## 背景

家里有台老笔记本最近安装了PVE，做ALL IN ONE测试。我希望在不同的机器上都可以使用copilot-service提供的服务。但是无奈没有一个稳定的公网IP，因此只能退而求其次，使用ZeroTier来做内网穿透服务。ZeroTier是通过绑定 /dev/net/tun 这个tun接口来进行组网的，然鹅，PVE下使用CT创建的LXC默认都没有这个接口，因此需要为它添加这个接口。

<!--more-->

## 解决方案

- ### 无特权容器

  - 修改pve主机（宿主机）的`/etc/pve/nodes/pve/lxc/101.conf`，101为LCX容器的CT ID.

    ```txt
    lxc.hook.autodev = sh -c "modprobe tun" 
    lxc.mount.entry=/dev/net/tun /var/lib/lxc/XXX/rootfs/dev/net/tun none bind,create=file
    ```

  - 重启LXC容器，就可以正常安装使用ZeroTier了

    ```shell
    pct reboot 101
    ```

- ### 特权容器

  这个还没有试过，等试过再更新。



参考链接：https://www.cnblogs.com/lynetwork/articles/17271495.html

