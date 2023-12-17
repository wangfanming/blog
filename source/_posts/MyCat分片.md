---
title: MyCat分片
date: '2019/3/28 10:43:32'
categories:
  - Web
tags:
  - MyCat
  - 分库分表
---


## MyCat简介
>&emsp;&emsp;Mycat 背后是阿里曾经开源的知名产品——Cobar。 Cobar的核心功能和优势是MySQL数据库分片。
&emsp;&emsp;Mycat 是基于 cobar 演变而来， 对 cobar 的代码进行了彻底的重构， 使用 NIO 重构了网络模块，并且优化了Buffer 内核， 增强了聚合，oin等基本特性， 同时兼容绝大多数数据库成为通用的数据库中间件。简单的说， MyCAT 就是： 一个新颖的数据库中间件产品支持 mysql 集群，或者mariadbcluster， 提供高可用性数据分片集群。你可以像使用 mysql 一样使用 mycat。 对于开
发人员来说根本感觉不到mycat的存在。

<!--more-->

## 环境需求

- JDK :  1.7及以上版本
- MySQL: mysql5.5以上版本

#### Mysql安装与启动步骤：

 1. 将MySQL的服务端和客户端安装包(RPM)上传到服务器
 2. 查询之前是否安装过MySQL
 ` # rpm -qa|grep mysql `
 3. 卸载旧版本的MySQL
 ` rpm -e --nodeps XXX`
 4. 安装服务端
 `rpm -ivh MySQL-server-5.6.17-1.el6.x86_64.rpm`
 5. 安装客户端
 `rpm -ivh MySQL-client-5.6.17-1.el6.x86_64.rpm`
 6. 启动MySQL服务
 `service mysql start`
 7. 登陆MySQL(刚安装的Mysql密码在`/root/.mysql_secret`)
`mysql -uroot -p密码`
 8. 设置远程登录权限
 `GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY '123456' WITH GRANT OPTION;`
  `flush privileges;`

#### MyCat分片

##### 分片相关的概念

- 逻辑库(schema) 
>&emsp;&emsp;前面一节讲了数据库中间件， 通常对实际应用来说， 并不需要知道中间件的存在， 业务开发
人员只需要知道数据库的概念， 所以数据库中间件可以被看做是一个或多个数据库集群构成
的逻辑库。(可以理解为一个保存了数据真实地址的映射集合)。

- 逻辑表(table)
>&emsp;&emsp;既然有逻辑库，那么就会有逻辑表，分布式数据库中，对应用来说，读写数据的表就是逻辑表。逻辑表，可以是数据切分后，分布在一个或多个分片库中，也可以不做数据切分，不分片，只有一个表构成。
&emsp;
**分片表**： 是指那些原有的很大数据的表，需要切分到多个数据库的表，这样， 每个分片都有一部分数据，所有分片构成了完整的数据。总而言之就是需要进行分片的表。
&emsp;
**非分片表**： 一个数据库中并不是所有的表都很大，某些表是可以不用进行切分的， 非分片是相对分片表来说的， 就是那些不需要进行数据切分的表。

- 分片节点(dateNode)
>&emsp;&emsp;数据切分后，一个大表被分到不同的分片数据库上面，每个表所在的数据库就是分片节点（dataNode)。

-节点主机(dateHost)
>&emsp;&emsp;数据分片后，每个分片节点(dateNode)不一定都会独占一台机器，同一台机器上可以有多个分片数据库，这样一个或多个分片节点所在的机器就是节点主机,为了规避单节点并发数限制，尽量将读写压力高的分片节点均衡放在不同的节点主机。

-分片规则(rule)
>&emsp;&emsp;一个大表被分成若干个分片表，就需要一定的规则，这样按照某种业务规则把数据分到某个分片的规则就是分片规则，数据切分选择选择合适的分片规则非常重要，将极大的避免后续数据处理的难度。

#### MyCat分片配置

##### 配置schema.xml

>schema 标签用于定义MyCat实例中的逻辑库
&emsp;
Table 标签定义了MyCat中的逻辑表 
&emsp;
rule 用于指定分片规则，auto-sharding-long 的分片规则是按ID值的范围进行分片,1-5000000为第1片,5000001-10000000 为第 2 片.... 
&emsp;
dataNode标签定义了MyCat中的数据节点，也就是数据分片。
dataHost标签在mycat逻辑库中也是作为最底层的标签存在，直接定义了具体的数据库实例、读写分离配置和心跳语句。
schema.xml配置如下：
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://org.opencloudb/">
<schema name="PINYOUGOUDB" checkSQLschema="false" sqlMaxLimit="100">
<table name="tb_test" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
</schema>
    <dataNode name="dn1" dataHost="localhost1" database="db1" />
    <dataNode name="dn2" dataHost="localhost1" database="db2" />
    <dataNode name="dn3" dataHost="localhost1" database="db3" />
    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
    writeType="0" dbType="mysql" dbDriver="native" switchType="1"
    slaveThreshold="100">
<heartbeat>select user()</heartbeat>

<writeHost host="hostM1" url="localhost:3306" user="root"
password="123456">
</writeHost>
</dataHost>
</mycat:schema>
```
##### 配置server.xml

server.xml几乎保存了mycat需要的系统配置信息。最常见的是在此配置用户名、密码以及权限。
server.xml配置如下：
添加UTF-8字符集设置，否则存储中文会出现问号

```xml
<property name="charset">utf8</property>
```
修改user的设置，为TESTDB添加root、test用户
```xml
<user name="test">
<property name="password">test</property>
<property name="schemas">TESTDB</property>
</user>
<user name="root">
<property name="password">123456</property>
<property name="schemas">TESTDB</property>
</user>
```
##### MyCat分片测试

进入逻辑库TESTDB
`# mysql -uroot -p123456 -P8066 -DTESTDB`
`show tables;`会发现已经有表`tb_test`
执行以下语句创建一个表

```sql
CREATE TABLE tb_test (
id BIGINT(20) NOT NULL,
title VARCHAR(100) NOT NULL ,
PRIMARY KEY (id)
) ENGINE=INNODB DEFAULT CHARSET=utf8
```
在创建完成后，会对表`td_test`的结构进行初始化。查看物理数据库`db1,db2,db3`,会发现，每个物理数据库中都已经创建好了表`tb_test`。

然后就可以向逻辑库中的逻辑表中插入数据，数据会按照分片规则进行分片存储。

##### MyCat分片规则

rule.xml用于定义分片规则

1. 按主键范围分片(rang-long)

```xml
    <tableRule name="auto-sharding-long">
        <rule>
            <columns>id</columns>
            <algorithm>rang-long</algorithm>
        </rule>
    </tableRule>
```

配置文件详解：
- tableRule :用于定义固体某个表或某类表的分片规则名称
- column：用于定义分片的列
- algorithm：代表算法的名称

**rang-long**的定义：

```js
    <function name="rang-long"
        class="org.opencloudb.route.function.AutoPartitionByLong">
        <property name="mapFile">autopartition-long.txt</property>
    </function>
    
```
 **autopartition-long.txt**的定义
```shell
    # range start-end ,data node index
    # K=1000,M=10000.
    0-500M=0
    500M-1000M=1
    1000M-1500M=2
```

----------

 2. 一致性哈希murmur
 当我们需要将数据平均分在几个分区中，需要使用一致性hash规则
```xml
<function name="murmur"
     class="org.opencloudb.route.function.PartitionByMurmurHash">
    <property name="seed">0</property><!--     默认是 0 -->
    <property name="count">3</property>
    <!--要分片的数据库节点数量，必须指定，否则没法分片-->
    <property name="virtualBucketTimes">160</property>
    <!-- 一个实际的数据库节点被映射为这么多虚拟节点， 默认是 160 倍，也就是虚拟节点数是物理节点数的 160 倍 -->
<!-- <property name="weightMapFile">weightMapFile</property> 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写， 以从 0 开始到 count-1的整数值也就是节点索引为key，以节点权重值为值。 所有权重值必须是正整数， 否则以 1 代替 -->
<!-- <property name="bucketMapPath">/etc/mycat/bucketMapPath</property>
用于测试时观察各物理节点与虚拟节点的分布情况， 如果指定了这个属性， 会把虚拟节点的 murmur hash 值与物理节点的映射按行输出到这个文件， 没有默认值， 如果不指定， 就不会输出任何东西 -->
</function>
```
配置文件中表规则的定义：
```xml
<tableRule name="sharding-by-murmur">
    <rule>
    <columns>id</columns>
    <algorithm>murmur</algorithm>
    </rule>
</tableRule>
```
在这里是按照id进行hash运算，然后确定数据的分片存储位置

