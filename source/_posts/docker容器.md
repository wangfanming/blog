# Docker

## 容器技术
> &emsp;&emsp;容器技术相比于管理程序虚拟化(hypervisor virtualization，HV)的不同之处在于，
管理程序虚拟化通过中间层将一台或者多台独立的机器虚拟运行与物理硬件之上，而容器则是直接运行在
操作系统内核之上的用户空间。因此，容器虚拟化也被称为“操作系统级虚拟化”，容器技术可以让多个
独立用户空间运行在同一台宿主机上。

> &emsp;&emsp;由于容器依赖于操作系统，采用的是**共享内核**的运行方式，所以容器只能运行与底层宿主机相同或者相似的操作系统。

> &emsp;&emsp;相对于彻底隔离的HV，容器被认为是不安全的。容器采用的是沙盒机制，其底层仍需
调用操作系统的接口来实现自身的功能，所以不能完全与宿主机系统完全隔离，会有潜在的风险。

> &emsp;&emsp; 尽管有诸多的局限性，容器还是被广泛部署于各种各样的应用场合。在差大规模的多
租户服务部署、轻量级沙盒以及对安全要求不太高的隔离环境中，容器技术非常流行。

> &emsp;&emsp;最新的容器技术引入了OpenVZ，Solaris Zones以及Linux容器(LXC)。使用这些新技
术，容器不再仅仅是一个单独的运行环境。在自己的权限内，容器更像一个完整的宿主机。对Dockers来
说，它得益于现代Linux特性，如控件组(control group)，命名空间(namespace)技术，容器和宿主机
之间的隔离更加彻底，容器有独立的网络和存储栈，还拥有自己的资源管理能力，使得同一台宿主机中的
多个容器可以友好的共存。

> &emsp;&emsp;和传统的虚拟化以及半虚拟化相比，容器不需要模拟层(emulation layer)和管理层
(hypervisor layer)，而是使用操作系统的系统调用接口，这降低了运行单个容器所需的开销，也使得
宿主机中可以运行更多的容器。

---
### 容器与虚拟机的比较
1、本质区别
>虚拟机依赖于**hypervisor**，在虚拟机使用硬件资源时，由hypervisor统一调度、分配，而容器是运行在
操作系统之上的一个隔离环境，可以认为是一个独立的进程，容器内的应用程序使用的其实是操作系统分
配给该容器的计算资源。

2、资源使用率
>相比于虚拟机，容器因为采用的是**共享内核**的工作方式，所以**容器可以看成是一个组装好了一套特定
应用程序的虚拟机，直接利用宿主机的内核，完成相应操作的虚拟机**。正式由于这个特性，使得容器抽象层
比虚拟机更少，更加轻量化，启动速度更快。但是带来的弊端也是显而易见的，共享内核的机制使得容器受制于
操作系统，并且安全性得不到足够的保证。而虚拟机则是牺牲一部分性能，获得的是完整的隔离性，使得虚拟机的
安全性大大提高，可移植性也更强。

| 对比项        | docker           | 虚拟化  |
| ------------|:--------------:|---------:|
| 快速创建、删除 | 启动应用         |  启动GuestOS + 启动应用 |
| 交付、部署     | 容器镜像         |   虚拟机镜像 |
| 单节点密度     | 100~1000        | 10~100     |
|更新管理        |迭代式更新，修改Dockerfile，对增量内容进行分发、存储、传输、节点启动和恢复迅速| 向虚拟机推送安装、升级应用软件补丁包  |
|稳定性          |每月更新一个版本  |KVM、Xen、VMware都已经很稳定|
|安全性          |Docker具有宿主机root权限|硬件隔离：Guest OS运行在非根模式|
|监控成熟程度    |还在发展中        |Host、Hypervisor,VM的监控工具在生产环境已使用多年|
|高可用性        |通过业务本身的高可用性来保证|快照、克隆、HA、动态迁移、异地容灾、异地双活|
|管理平台成熟度  |以k8s为代表，还在快速发展过程中|以Openstack、vCenter、OPV-Suite为代表，已经在生产环境使用多年|

---
## Docker特点
1、上手快

&emsp;&emsp;Docker依赖于“写时复制”(copy-on-write)模型，使得修改应用很快。然后就可以创建容器来运行应用程序。

2、职责的逻辑分类

&emsp;&emsp;使用Docker，开发人员只需关心容器内的应用程序，而运维人员只需要关系如何管理容器。Docker加强了开发人员写代码时的
开发环境与应用程序部署的生产环境的一致性。降低了“开发时一切正常，上线出问题”的环境问题发生概率。

3、快速高效的开发生命周期

&emsp;&emsp;缩短代码从开发、 测试到部署、 上线运行的周期， 让应用程序具备可移植性， 易于构建， 并易于协作。

4、鼓励使用面向服务的架构

&emsp;&emsp;Docker推荐单个容器只运行一个应用程序或进程，这样就形成了一个分布式的应用程序模型。在这种模型下，应用程序
或者服务都可以表示为一系列内部互联的容器，从而使得分布式部署应用程序，扩展或调试应用程序都变得简单，同时也提高了程序的内省性。

## Docker组件

##### Docker客户端和服务器

&emsp;&emsp;Docker是一个C/S架构程序。Docker客户端只需要Docker服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作
并返回结果。Docker提供了一个命令行工具Docker以及一套RESTful API。

##### Docker镜像

&emsp;&emsp;镜像是构建Docker的基石。镜像是基于联合文件系统的一种层式结构，有一系列指令一步步构建出来。镜像体积很小，非常“便捷”，
易于分享、存储和更新。

##### Registry(注册中心)

&emsp;&emsp;Docker用Registry来保存用户构建的镜像。Registry分为公共和私有两种。Docker公司运营公共的Registry叫做Docker Hub。

##### Docker容器

&emsp;&emsp;Docker可以帮助你构建和部署容器，我们只需把自己的应用程序或者服务打包进容器即可。容器是基于镜像启动起来的，容器中
可以运行一个或多个进程。我们可以认为，镜像是Docker生命周期中的构建或者打包阶段，而容器则是启动或者执行阶段。容器基于景象启动，
一旦容器启动完成，我们就可以登录到容器中安装自己需要的软件或者服务。

## Docker可以做什么

- 加速本地开发和构建流程，使其更加高效，更加轻量化。本地人员可以构建、运行并分享Docker容器。容器可以在开发环境中构建，然后提交到测试环境，
并最终部署至生产环境。

- 能够让独立的服务或应用程序在不同的环境中，得到相同的运行结果。这一点在面向服务和重度依赖微服务的部署尤为适用。

- 用docker创建隔离的环境来进行测试。例如，用Jenkins CI这样的持续集成工具启动一个用于测试的容器。

- Docker 可以让开发者现在本机上构建一个复杂的程序或者架构来进行测试，而不是一开始就在生产环境部署、测试。

---

## Docker安装与启动

#### 安装Docker

1、使用yum命令在线安装
>`yum install docker`

2、查看docker版本
>`docker -v`

#### 启动与停止docker

- 启动docker:  `systemctl start docker`

- 停止docker:  `systemctl stop docker`

- 重启docker: `systemctl restart docker`

- 查看docker状态：`systemctl status docker`

- 开机启动： `systemctl enable docker`

**systemctl命令是系统服务管理器指令，是service和chkconfig两个命令的组合**

#### Docker镜像的结构
![](https://github.com/wangfanming/wangfanming.GitHub.io/blob/master/image/docker镜像的结构.bmp)

#### 列出镜像

列出docker下的所有镜像：`docker images`

![](https://github.com/wangfanming/wangfanming.GitHub.io/blob/master/image/列出docker下的镜像信息.bmp)

- REPOSITORY： 镜像所在的仓库名称

- TAG： 镜像标签(可以直观地理解为镜像的版本号)

- IMAGE ID： 镜像 ID(镜像的唯一标识)

- CREATED： 镜像的创建日期（不是获取该镜像的日期，是镜像在构建成功时的时间）

-  SIZE： 镜像大小

这些镜像都存储在宿主机的/var/lib/docker目录下。

>我们在运行同一个仓库中的不同镜像时， 可以通过在仓库名后面加上一个冒号和标签名
来指定该仓库中的某一具体的镜像， 例如 `docker run --name custom_container_name –i –t
docker.io/ubunto:12.04 /bin/bash`， 表明从镜像 Ubuntu:12.04 启动一个容器， 而这个镜像的操
作系统就是 Ubuntu:12.04。 在构建容器时指定仓库的标签也是一个好习惯。

#### 配置Dcoker的镜像源

1、编辑`/etc/docker/daemon.json`,不存在就手动创建。

2、在该文件中输入如下内容：
>`{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
  }`

3、重启docker服务

#### 拉取、搜索、删除镜像
1、拉取镜像

`docker pull 镜像名称`

2、搜索镜像
`docker search 镜像名称` ，例如：`docker search tomcat`

![](https://github.com/wangfanming/wangfanming.GitHub.io/blob/master/image/搜索镜像.bmp)

- NAME： 仓库名称
- DESCRIPTION： 镜像描述
- STARS： 用户评价， 反应一个镜像的受欢迎程度
- OFFICIAL： 是否官方
- AUTOMATED： 自动构建， 表示该镜像由 Docker Hub 自动构建流程创建的

3、删除
- 删除指定镜像
`docker rmi 镜像ID`

- 删除所有镜像
>
```shell
    docker rmi `docker image -q`
```

## Docker容器操作

1、创建容器

&emsp;&emsp;命令:`docker run`

&emsp;&emsp;命令参数说明：

- `-i`: 表示以“交互模式”运行容器（类似于redis的前端启动）
- `-t`:表示容器启动后进入其命令行。加入这两个参数后，容器创建就能登陆进去，即分配
一个伪终端。
- `--name` :为创建的容器命名。
- `-v`:表示目录映射关系(前者是宿主机目录，后者是容器中映射到宿主机上的目录)，
可以使用多个`-v`做多个目录或文件映射。**注意：最好做目录映射，在宿主机上做修改，然后共享到容器上**。
- `-d`:在`run`后边加上`-d`参数，则会创建一个守护式容器在后台运行。（类似于redis的后端启动）
- `-p`:表示端口映射，前者是宿主机端口，后者是容器内的映射端口。也可以像`-v`一样做多个端口映射。

2、查看容器
- 查看所有的容器(历史启动过的容器)：`docker ps -a`
- 查看最后一次运行的容器：`docker ps -l`
- 查看正在运行的容器：`docker ps`

3、停止与启动容器
- 停止正在运行的容器：`docker stop $CONTAINER_NAME/ID`
- 启动已运行过的容器：`docker start $CONTAINER_NAME/ID`

4、删除容器
- 删除指定的容器：`docker rm $CONTAINER_ID/NAME`
- 删除所有容器：
>
```shell
docker rm `docker ps -a -q`
```

## Docker部署应用(示例)

1、拉取MySQL镜像
```shell
    docker pull mysql
```
2、创建MySQL容器
```shell
docker run --name pinyougoou_mysql -p 33306:3306 -e MYSQL_ROOT_PASSWORD=123456
-di mysql
```
3、进入MySQL容器，并登录

进入容器
```shell
docker exec -it pinyougou_mysql /bin/bash
```
登陆MySQL
```shell
mysql -uroot -p
```

4、远程登陆MySQL

原理:利用容器启动时的端口映射，我们只需要在Navicat上连接虚拟机的`33306`端口，
即可映射至容器内的3306端口，连接上容器内的MySQL，然后就可以像操作普通MySQL数据库
一样，对容器内MySQL数据库进行操作。

![](https://github.com/wangfanming/wangfanming.GitHub.io/blob/master/image/连接容器内mysql.bmp)

5、查看容器IP地址

以下命令可以查看容器运行的各种数据：

`docker inspect pinyougou_mysql`

![](https://github.com/wangfanming/wangfanming.GitHub.io/blob/master/image/查看容器运行情况.bmp)

也可以使用以下命令直接输出IP地址：

`docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql_pinyougou`

**这个IP地址是由docker的服务端维护的，可以用于该服务端下所有客户端的通信**。

---
## 备份与迁移

#### 将容器保存成镜像
>
```shell
docker commit pinyougou_mysql myMysql
```
- pinyougou_mysql 是容器名称
- myMysql是新的镜像名称

#### 镜像备份
>
```shell
docker save -o /home/myMysql.tar myMysql
```

- `-o` ：指定了`myMysql`镜像的备份文件的位置

#### 镜像的恢复与迁移

>
```shell
docker load -i /home/myMysql.tar
```
- `-i`：加载的备份文件的路径


















