---
title: Nginx的安装与静态网站部署
date: '2017/6/18 20:51:31'
categories:
  - Web
tags:
  - nginx
  - 反向代理
  - 负载均衡
---



## Nginx简介

>&emsp;&emsp;Nginx是一款高性能的HTTP服务器/反向代理服务器及电子邮件服务器。
官方测试能够支撑5万并发访问，并且CPU、内存等资源消耗却非常低，运行非常稳定。

### Nginx应用场景
- HTTP服务器。Nginx可以做网页静态服务器。
- 虚拟主机。 可以实现在一台服务器虚拟出多个网站。
- 反向代理，负载均衡。当单台服务器不能满足用户的请求时，需要用多台服务器集群可以使用
nginx做反向代理。并且多台服务器可以平均分担负载，不会因为某台服务器负载过高宕机
而某台服务器闲置的情况。

<!--More-->

### Nginx在Linux下的安装
环境：
- gcc-c++，pcre，pcre-devel，zlib，zlib-devel，openssl，openssl-devel

1、上传并解压缩
>`tar zxvf nginx-1.8.0.tar.gz`

2、进入nginx-1.8.0目录，使用configure命令创建MakeFile文件
```shell
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi

```

configure 参数详解：
```shell
./configure \
--prefix=/usr \ 指向安装目录
--sbin-path=/usr/sbin/nginx \ 指向（执行） 程序文件（nginx）
--conf-path=/etc/nginx/nginx.conf \ 指向配置文件
--error-log-path=/var/log/nginx/error.log \ 指向 log
--http-log-path=/var/log/nginx/access.log \ 指向 http-log
--pid-path=/var/run/nginx/nginx.pid \ 指向 pid
--lock-path=/var/lock/nginx.lock \ （安装文件锁定， 防止安装文件被别人利用， 或自己
误操作。 ）
--user=nginx \
--group=nginx \
--with-http_ssl_module \ 启用 ngx_http_ssl_module 支持（使支持 https 请求， 需已安装
openssl）
--with-http_flv_module \ 启用 ngx_http_flv_module 支持（提供寻求内存使用基于时间的
偏移量文件）
--with-http_stub_status_module \ 启用 ngx_http_stub_status_module 支持（获取 nginx 自上次启
动以来的工作状态）
--with-http_gzip_static_module \ 启用 ngx_http_gzip_static_module 支持（在线实时压缩输出数据
流）
--http-client-body-temp-path=/var/tmp/nginx/client/ \ 设定 http 客户端请求临时文件路径
--http-proxy-temp-path=/var/tmp/nginx/proxy/ \ 设定 http 代理临时文件路径
--http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \ 设定 http fastcgi 临时文件路径
--http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \ 设定 http uwsgi 临时文件路径
--http-scgi-temp-path=/var/tmp/nginx/scgi \ 设定 http scgi 临时文件路径
--with-pcre 启用 pcre 库
```
3、编译
>`make -j 8 `
参数 `-j` 用于指定编译的线程的个数。

4、安装
>`make install`

5、创建启动参数需要的文件目录`/var/temp/nginx/client`
>`mkdir /var/temp/nginx/client -p`

**Nginx的相关命令**
- 启动nginx :`./nginx`
- 关闭nginx :`./nginx -s quit`
- 重启nginx :`./nginx -s reload`

## Nginx虚拟主机的配置
1、上传静态网站
>将前端静态页cart.html以及图片样式等资源，上传至 /usr/local/nginx/cart

2、修改Nginx的修改配置文件：/usr/local/nginx/conf/nginx.conf
```json
server {
    listen 81; # 对外访问端口
    server_name localhost;

    location / {

        root cart;
        index cart.html;

        }
    error_page 500 502 503 504 /50x.html;
    location = /50x.html{
        root html;
        }
    }
```
3、域名绑定

3.1 修改nginx.conf
```json
server {
    listen 81; # 对外访问端口
    server_name cart.pinyougou.com;

    location / {

        root cart;
        index cart.html;

        }
    error_page 500 502 503 504 /50x.html;
    location = /50x.html{
        root html;
        }
    }
```
3.2、修改本机hosts映射
```shell
127.0.0.1 cart.pinyougou.com
```
---

## Nginx反向代理与负载均衡

> 反向代理(Reverse Proxy):是指以**代理服务器**来接受Internet上的连接请求，然后将请求转发给内部网络
上的服务器，并将从服务器上得到的结果返回给Internet上的请求连接的客户端，此时代理服务器对外就表现
为一个反向代理服务器。正向代理针对的是客户端，而反向代理针对的是服务器。

### 配置反反向代理
1、在Nginx主机修改Nginx配置文件
```shell
upstream tomcat-cart{
    server 192.168.25.135:9102;
}

server {
    listen 80;
    server_name cart.pinyougou.com;

    location / {
        proxy_pass http://tomcat-cart;
        index index.html index.htm;
    }
}
```
2、配置域名解析

3、重启nginx,然后使用域名访问
>`http://cart.pinyougou.com/cart/index.html`

---

# 负载均衡
>&emsp;&emsp; 负载均衡(Load Balance),其意思就是分摊到多个操作单元上进行执行，例如
Web服务器、FTP服务器、企业关键应用服务器和其他关键服务器等，从而共同完成工作任务。
它建立在现有网络结构上，提供了一种**廉价有效透明的方法扩展网络设备**和**服务器带宽**、**增加吞吐量**
**加强网络数据处理能力**、**加强网络数据处理能力**、**提高网络的灵活性和可用性**。

## 负载均衡的配置(示例)
1、部署两个tomcat服务，端口分别为9801和9802.

2、部署网站首页
```shell
[root@localhost ~]# cp -r portal/* /usr/local/tomcat-cluster/tomcat-portal-1/webapps/ROOT
[root@localhost ~]# cp -r portal/* /usr/local/tomcat-cluster/tomcat-portal-2/webapps/ROOT
```
3、配置负载均衡

3.1、设置域名指向
`192.168.25.135 www.pinyougou.com`

3.2、修改Nginx配置文件：
```shell
upstream tomcat-portal{
    #配置服务列表，轮询调度以实现负载均衡,**weight**设置服务被调用的权重
    server 192.168.25.135:9801 weight=2;
    server 192.168.25.135:9802;
}

server {
    listen 80;
    server_name www.pinyougou.com;
    location / {
        proxy_pass http://tomcat-portal;
        index index.html;
    }
}
```





