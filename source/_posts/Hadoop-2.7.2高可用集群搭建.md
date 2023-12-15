---
title: Hadoop-2.7.2高可用集群搭建(Ubuntu14.04)
date: '2020/8/31 08:53:31'
categories:
  - 大数据
tags:
  - hadoop
  - HA
---

**以root用户身份进行配置**

### 环境需求：
- JDK版本：1.7+
- Hadoop版本：hadoop-2.7.2
- zookeeper版本：zookeeper-3.4.5

#### 详细配置

| 主机名IP      | Namenode         | Datanode  | Yarn   | Zookeeper |JournalNode |
| ------------|:--------------:|---------:|------:|---------:|----------:|
| node-1:192.168.111.130 | 是      | 是        |否      | 是        |  是        |
| node-2:192.168.111.129 | 是      | 是        |是      | 是        |  是        |
| node-3:192.168.111.128 | 否      | 是        |是      | 是        |  是        |

1、安装JDK

 (1)  下载 jdk-7u80-linux-x64.tar 文件

 (2)  解压缩文件 tar -zxvf jdk-7u80-linux-x64.tar(尽量使用jdk1.7)

 (3)  在 /etc/profile文件末尾，添加jdk的配置信息

 (4) 重新加载文件，使配置生效：#source /etc/profile

 2、配置SSH免密登陆
 >SSH免密钥原理：
      我们使用ssh-keygen在ServerA上生成私钥跟公钥，将生成的公钥拷贝到远程机器ServerB上后,就可以使用ssh命令无需密码登录到另外一台机器ServerB上。生成公钥与私钥有两种加密方式，第一种是rsa(默认)，还有一种是dsa,使用时两种方式随便选一种即可

  (1) 安装SSH服务
  ```shell
    # apt-get install ssh
  ```

  (2) 在三个节点上分别都使用rsa加密生成密钥对
  ```shell
    # ssh-keygen -t rsa
  ```
   按3次Enter键后，会在/root/.ssh/下生成id_rsa (私钥)，和id_rsa.pub(公钥)

  (3) 将node-1上，将id_rsa.pub读取到authorized_keys中(authorized_keys 第一次读入是不存在的)
  ```shell
    # cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  ```

  (4) 分别将node-2和node-3上的公钥都读入到master的authorized_keys中

       在node-2和node-3上：
            #scp /root/.ssh/id_rsa.pub  master:/root/
       在node-1上：
       每收到一个公钥都读取一次(一个一个地写入文件)：
            # cat /root/id_rsa.pub >> /root/.ssh/authorized_keys

  (5) 将node-1上的authorized_keys发送到node-2和node-3，以实现三个节点之间的免密钥登录
  ```shell
    #scp /root/.ssh/authorized_keys   node-2:/root/.ssh/
    #scp /root/.ssh/authorized_keys   node-3:/root/.ssh/
  ```

  (6) 在3个节点上修改authorized_keys的权限：
  ```shell
    # chmod 600 ~/.ssh/authorized_keys
  ```

 3、Zookeeper集群搭建

  (1) 下载zookeeper-3.4.9.tar.gz

  (2)  解压
   ```shell
    #tar -zxvf zookeeper-3.4.9.tar.gz -C /home/
   ```
  (3) 修改zoo.cfg（可以使用zoo_sample.cfg来生成zoo.cfg）
   ```shell
     # cp zoo_sample.cfg zoo.cfg
     # vim zoo.cfg
   ```

   **参数说明**:

  - tickTime: zookeeper中使用的基本时间单位, 毫秒值.
  - dataDir: 数据目录. 可以是任意目录.
  - dataLogDir: log目录, 同样可以是任意目录. 如果没有设置该参数, 将使用和dataDir相同的设置.
  - clientPort: 监听client连接的端口号.
  - server.X=A:B:C 其中X是一个数字, 表示这是第几号server. A是该server所在的IP地址. B配置该server和集群中的leader交换消息所使用的端口. C配置选举leader时所使用的端口. 由于配置的是伪集群模式, 所以各个server的B, C参数必须不同.

  (4). 根据zoo.cfg各自在节点上的 dataDir下添加myid

        node-1：
                #echo "1" >> /home/zookeeperData/myid
        node-2:
                #echo "2" >> /home/zookeeperData/myid
        node-3:
                #echo "3" >> /home/zookeeperData/myid

  (5) 在各个节点上开启zookeeper服务,并且查看服务是否成功启动
   ```shell
      #zkServer.sh start
      #zkServer.sh status
   ```

4、Hadoop集群的搭建

   (1) 下载 hadoop-2.7.2.tar.gz,并解压
   ```shell
      tar -zxvf  hadoop-2.7.2.tar.gz -C /home/
   ```
   (2)修改如下配置文件(在/home/hadoop-2.7.2/etc/hadoop/下)

   **修改core-site.xml**
   ```xml
   <configuration>
      <property>
            <name>fs.defaultFS</name>
            <value>hdfs://nameserver</value>
       </property>
      <property>
            <name>hadoop.tmp.dir</name>
            <value>/home/hdfs_all/tmp/</value>
       </property>
      <property>
            <name>io.file.buffer.size</name>
            <value>1024</value>
       </property>
      <property>
            <name>ha.zookeeper.quorum</name>
            <value>node-1:2181,node-2:2181,node-3:2181</value>
       </property>
    </configuration>
   ```
   **修改hdfs-site.xml**
   ```xml
   <configuration>
       <property>
               <name>dfs.nameservices</name>
               <value>nameserver</value>
       </property>
       <property>
              <name>dfs.ha.namenodes.nameserver</name>
              <value>nn1,nn2</value>
       </property>
       <property>
              <name>dfs.namenode.rpc-address.nameserver.nn1</name>
              <value>node-1:9000</value>
       </property>
       <property>
               <name>dfs.namenode.http-address.nameserver.nn1</name>
               <value>node-1:50070</value>
           </property>
       <property>
              <name>dfs.namenode.rpc-address.nameserver.nn2</name>
              <value>node-2:9000</value>
       </property>
       <property>
               <name>dfs.namenode.http-address.nameserver.nn2</name>
               <value>node-2:50070</value>
           </property>
       <property>
                <name>dfs.namenode.shared.edits.dir</name>
                <value>qjournal://node-1:8485;node-2:8485;node-3:8485/nameserver</value>
           </property>
       <property>
       <name>dfs.journalnode.edits.dir</name>
                 <value>/home/hdfs_all/journal</value>
           </property>
       <property>
                 <name>dfs.ha.automatic-failover.enabled</name>
                 <value>true</value>
           </property>
       <property>
                   <name>dfs.client.failover.proxy.provider.nameserver</name>
                   <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
           </property>
       <property>
                    <name>dfs.ha.fencing.methods</name>
                    <value>sshfence</value>
           </property>
       <property>
                   <name>dfs.ha.fencing.ssh.private-key-files</name>
                   <value>/root/.ssh/id_rsa</value>
           </property>
       <property>
               <name>dfs.namenode.name.dir</name>
               <value>file:///home/hdfs_all/hdfs/name</value>
           </property>
       <property>
               <name>dfs.datanode.data.dir</name>
               <value>file:///home/hdfs_all/hdfs/data</value>
           </property>
       <property>
              <name>dfs.replication</name>
              <value>2</value>
        </property>
   </configuration>
   ```

   **修改mapred-site.xml**(刚解压的文件中没有给文件，将mapred-site-tempalte.xml复制一份即可)
   ```xml
   <configuration>
       <property>
               <name>mapreduce.framework.name</name>
               <value>yarn</value>
       </property>
   </configuration>
   ```
   **修改yarn-site.xml**

```xml
    
       <configuration>
       <!-- 开启RM高可用 -->
       <property>
       <name>yarn.resourcemanager.ha.enabled</name>
       <value>true</value>
       </property>
       <!-- 指定RM的cluster id -->
       <property>
       <name>yarn.resourcemanager.cluster-id</name>
       <value>yrc</value>
       </property>
       <!-- 指定RM的名字 -->
       <property>
       <name>yarn.resourcemanager.ha.rm-ids</name>
       <value>rm1,rm2</value>
       </property>
       <!-- 分别指定RM的地址 -->
       <property>
       <name>yarn.resourcemanager.hostname.rm1</name>
       <value>node-2</value>
       </property>
       <property>
       <name>yarn.resourcemanager.hostname.rm2</name>
       <value>node-3</value>
       </property>
       <!-- 指定zk集群地址 -->
       <property>
       <name>yarn.resourcemanager.zk-address</name>
       <value>node-1:2181,node-2:2181,node-3:2181</value>
       </property>
       <property>
       <name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
       </property>
       </configuration>
       
```

**修改slaves**

  删除`localhost`,添加如下内容：
  ```shell
  node-1
  node-2
  node-3
  ```
**修改JAVA_HOME**

分别在文件hadoop-env.sh和yarn-env.sh中添加JAVA_HOME配置

![](/images/java_home.bmp)

5、将hadoop配置文件发送至另外两个节点
```shell
scp -r /home/hadoop-2.7.2  node-2:/home/
scp -r /home/hadoop-2.7.2  node-3:/home/
```
6、给Hadoop配置环境变量(完整的环境变量配置如下)

![](/images/环境变量.bmp)



7、将环境变量发送给另外两个节点

```shell
scp -r /etc/profile  node-2:/etc
scp -r /etc/profile  node-3:/etc
```

**在发送完成后，都要使用`source /etc/profile`命令重新加载环境变量**

8、启动集群

**按顺序执行以下操作**

在node-1上：
```shell
zkServer.sh start
zkServer.sh status(要确保zookeeper服务已经启动)
hadoop-daemons.sh start journalnode   (启动journalnode服务)
hdfs zkfc –formatZK(格式化zkfc,让在zookeeper中生成ha节点,只需在第一次启动集群是格式化即可)

hadoop namenode -format (格式化hdfs,也只在第一次启动集群时格式化)
hadoop-daemon.sh start namenode

```
在node-2上(设置node-2为备份的namenode,所以需要在node-2上启动namenode进程)：
```shell
hdfs namenode -bootstrapStandby(在第一次启动时执行)
hadoop-daemon.sh start namenode
```
启动datanode(任意节点都可以):
```shell
hadoop-daemons.sh start datanode
```
在node-3上启动yarn；
```shell
start-yarn.sh
```
启动zkfc服务(任意节点都可以):
```shell
hadoop-daemons.sh start zkfc
```



然后就可以Web进行访问了

http://node-1:50070

http://node-2:50070



