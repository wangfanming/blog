---
title: LCX中运行copilot-service
categories:
  - 虚拟化
tags:
  - LCX

---

# LCX中运行copilot-service

## 简介

​		LXC 是一个知名的 **Linux 容器运行时**，包括各类工具、模板、库以及语言绑定。 它非常底层，相当灵活，并覆盖了上游内核支持的几乎所有与容器相关的功能。copilot-gpt4-service用来帮助我们转发对ChatGPT4的请求。本文记录里怎么在PVE下的LCX中运行copilot-service。

<!--more-->

## 步骤

1. 选择一个linux系统CT作为基础环境，如ubuntu

2. 下载copilot-gpt4-service源码

   ```shell
   git clone https://gitee.com/big-datasource/copilot-gpt4-service.git
   ```

3. 安装golang，并配置环境变量。

4. 配置golang的国内代理：

   ```shell
   go env -w GO111MODULE=on
   go env -w GOPROXY=https://goproxy.cn,direct
   ```

   经测试，启动这个服务，必须使用七牛云的代理，阿里云不行。

5. 为copilot-gpt4-service配置开机启动项

   1. 在/etc/systemd/system目录下新建copilot.service，内容如下：

      ```shell
      [Unit]
      Description=copilot-gpt4-service
      After=network.target
       
      [Service]
      Type=simple
      ExecStart=/opt/go/bin/go run /root/copilot-gpt4-service
      WorkingDirectory=/root/copilot-gpt4-service
      User=root
      Group=root
      Restart=always
      
      [Install]
      WantedBy=multi-user.target
      ```

   2. 配置开机启动项

      ```shell
      sudo systemctl daemon-reload
      sudo systemctl enable copilot.service
      sudo systemctl start copilot.service
      ```

   3. 使用`journalctl -u copilot.service`命令 即可查看服务的运行日志