---
title: Hive SQL编译原理与数据倾斜处理
date: '2023/4/28 10:46:32'
categories:
  - 数据治理
  - 数据仓库
tags:
  - Hive

---

# Hive SQL编译原理与数据倾斜处理

## HiveSQL编译成MapReduce的过程

![hivesql编译过程](images/hivesql编译过程.png)

1. 词法、语法解析: Antlr 定义 SQL 的语法规则，完成 SQL 词法，语法解析，将 SQL 转化为抽象语法树 AST Tree；
1. 语义解析: 遍历 AST Tree，抽象出查询的基本组成单元 QueryBlock；
2. 生成逻辑执行计划: 遍历 QueryBlock，翻译为执行操作树 OperatorTree；
3. 优化逻辑执行计划: 逻辑层优化器进行 OperatorTree 变换，合并 Operator，达到减少 MapReduce Job，减少数据传输及 shuffle 数据量；
4. 生成物理执行计划: 遍历 OperatorTree，翻译为 MapReduce 任务；
5. 优化物理执行计划: 物理层优化器进行 MapReduce 任务的变换，生成最终的执行计划。

<!--more-->

**Join的实现原理**

以下面这个SQL为例，讲解 join 的实现：

```
select u.name, o.orderid from order o join user u on o.uid = u.uid;
```

在map的输出value中为不同表的数据打上tag标记，在reduce阶段根据tag判断数据来源。MapReduce的过程如下：

![join过程](images/join过程.png)

**Group By的实现原理**

以下面这个SQL为例，讲解 group by 的实现：

```
select rank, isonline, count(*) from city group by rank, isonline;
```

将GroupBy的字段组合为map的输出key值，利用MapReduce的排序，在reduce阶段保存LastKey区分不同的key。MapReduce的过程如下:

![Image](images/groupby过程.png)

**Distinct的实现原理**

以下面这个SQL为例，讲解 distinct 的实现：

```
select dealid, count(distinct uid) num from order group by dealid;
```

当只有一个distinct字段时，如果不考虑Map阶段的Hash GroupBy，只需要将GroupBy字段和Distinct字段组合为map输出key，利用mapreduce的排序，同时将GroupBy字段作为reduce的key，在reduce阶段保存LastKey即可完成去重:

![](images/Distinct过程.png)



## 数据倾斜问题剖析

原则：没有瓶颈谈优化，毫无意义。

在分布式系统中，数据倾斜现象时不可避免的。但是发生了数据倾斜要不要处理，要怎么处理，这是个问题。单机资源足够大，数据量不大，发生了倾斜我们也难以感知。只有当数据量超出了某单机的处理能力时，才会影响我们的业务。此时，就需要我们对数据倾斜进行分析处理。

从本质上讲，发生数据倾斜的情况有两种：

1. map段读取到了不可分割的大文件，某个文件太大，超出了task的处理能力
2. shuffle阶段由于数据分布不均匀，导致reduce接收到的数据出现倾斜。

### 解决方案

1. **空值引发的数据倾斜**

   ​	实际业务中有些大量的null值或者一些无意义的数据参与到计算作业中，表中有大量的null值，如果表之间进行join操作，就会有shuffle产生，这样所有的null值都会被分配到一个reduce中，必然产生数据倾斜。

   ​	如果A、B两表join操作，假如A表中需要join的字段为null，但是B表中需要join的字段不为null，这两个字段根本就join不上，为什么还会放到一个reduce中呢？

   ​	这是因为这个倾斜发生的根本原因在于shuffle动作，只要key的hash结果是一样的，它们在进行shuffle后就会被拉到同一个reduce中。

   解决办法两种：

   1. 可以直接不让null值参与join操作，即不让null值有shuffle阶段

      ```sql
      SELECT *
      FROM log a
       JOIN users b
       ON a.user_id IS NOT NULL
        AND a.user_id = b.user_id
      UNION ALL
      SELECT *
      FROM log a
      WHERE a.user_id IS NULL;
      ```

   2. 因为null值参与shuffle时的hash结果是一样的，那么我们可以给null值随机赋值，这样它们的hash结果就不一样，就会进到不同的reduce中

      ```sql
      SELECT *
      FROM log a
       LEFT JOIN users b ON CASE 
         WHEN a.user_id IS NULL THEN concat('hive_', rand())
         ELSE a.user_id
        END = b.user_id;
      ```

2. **不同数据类型引发的数据倾斜**

   ​	对于两个表join，表a中需要join的字段key为int，表b中key字段既有string类型也有int类型。当按照key进行两个表的join操作时，默认的Hash操作会按int型的id来进行分配，这样所有的string类型都被分配成同一个id，结果就是所有的string类型的字段进入到一个reduce中，引发数据倾斜。

   解决办法：

   ​	如果key字段既有string类型也有int类型，默认的hash就都会按int类型来分配，那我们直接把int类型都转为string就好了，这样key字段都为string，hash时就按照string类型分配了：

   ```sql
   SELECT *
   FROM users a
    LEFT JOIN logs b ON a.usr_id = CAST(b.user_id AS string);
   ```

3. **不可拆分大文件引发的数据倾斜**

   ​	如果文件是因为采用了不支持分割操作的压缩方式，Map任务读取时只能整个读取，不会按照Block划分。如果这个压缩文件很大，则会导致Map端数据倾斜。

   解决办法：

   ​	在数据存储时使用bzip和zip等可分割的压缩算法。

4. **数据膨胀引发的数据倾斜**

   ​	在多维聚合计算时，如果进行分组聚合的字段过多，如下：

   ```sql
   select a，b，c，count（1）from log group by a，b，c with rollup;	
   ```

   > 注：with rollup是用来在分组统计数据的基础上再进行统计汇总，即用来得到group by的汇总信息。

   ​	如果上面的log表的数据量很大，并且Map端的聚合不能很好地起到数据压缩的情况下，会导致Map端产出的数据急速膨胀，这种情况容易导致作业内存溢出的异常。如果log表含有数据倾斜key，会加剧Shuffle过程的数据倾斜。

   解决方案：

   可以拆分上面的sql，将with rollup拆分成如下几个sql：

   ```sql
   SELECT a, b, c, COUNT(1)
   FROM log
   GROUP BY a, b, c;
   
   SELECT a, b, NULL, COUNT(1)
   FROM log
   GROUP BY a, b;
   
   SELECT a, NULL, NULL, COUNT(1)
   FROM log
   GROUP BY a;
   
   SELECT NULL, NULL, NULL, COUNT(1)
   FROM log;
   ```

   ​	但是，上面这种方式不太好，因为现在是对3个字段进行分组聚合，那如果是5个或者10个字段呢，那么需要拆解的SQL语句会更多。

   ​	在Hive中可以通过参数 hive.new.job.grouping.set.cardinality 配置的方式自动控制作业的拆解，该参数默认值是30。表示针对grouping sets/rollups/cubes这类多维聚合的操作，如果最后拆解的键组合大于该值，会启用新的任务去处理大于该值之外的组合。如果在处理数据时，某个分组聚合的列有较大的倾斜，可以适当调小该值。

5. **表连接时引发的数据倾斜**

   ​	两表进行普通的repartition join时，如果表连接的键存在倾斜，那么在 Shuffle 阶段必然会引起数据倾斜。

   解决方案：

   ​	通常做法是将倾斜的数据存到分布式缓存中，分发到各个 Map任务所在节点。在Map阶段完成join操作，即MapJoin，这避免了 Shuffle，从而避免了数据倾斜。

   > MapJoin是Hive的一种优化操作，其适用于小表JOIN大表的场景，由于表的JOIN操作是在Map端且在内存进行的，所以其并不需要启动Reduce任务也就不需要经过shuffle阶段，从而能在一定程度上节省资源提高JOIN效率。

   ​	在Hive 0.11版本之前，如果想在Map阶段完成join操作，必须使用MAPJOIN来标记显示地启动该优化操作，由于其需要将小表加载进内存所以要注意小表的大小。

   如将a表放到Map端内存中执行，在Hive 0.11版本之前需要这样写：

   ```sql
   select /* +mapjoin(a) */ a.id , a.name, b.age 
   from a join b 
   on a.id = b.id;
   ```

   ​	如果想将多个表放到Map端内存中，只需在mapjoin()中写多个表名称即可，用逗号分隔，如将a表和c表放到Map端内存中，则 / *+mapjoin(a,c)* / 。

   在Hive 0.11版本及之后，Hive默认启动该优化，也就是不在需要显示的使用MAPJOIN标记，其会在必要的时候触发该优化操作将普通JOIN转换成MapJoin，可以通过以下两个属性来设置该优化的触发时机：

   - hive.auto.convert.join=true 默认值为true，自动开启MAPJOIN优化。

   - hive.mapjoin.smalltable.filesize=2500000 默认值为2500000(25M)，通过配置该属性来确定使用该优化的表的大小，如果表的大小小于此值就会被加载进内存中。

   注意：使用默认启动该优化的方式如果出现莫名其妙的BUG(比如MAPJOIN并不起作用)，就将以下两个属性置为fase手动使用MAPJOIN标记来启动该优化:

   - hive.auto.convert.join=false (关闭自动MAPJOIN转换操作)

   - hive.ignore.mapjoin.hint=false (不忽略MAPJOIN标记)

   再提一句：将表放到Map端内存时，如果节点的内存很大，但还是出现内存溢出的情况，我们可以通过这个参数 mapreduce.map.memory.mb 调节Map端内存的大小。

6. 确实无法减少数据量引发的数据倾斜

   ​	在一些操作中，如在使用 collect_list 函数时，我们没有办法减少数据量

   ```sql
   select s_age,collect_list(s_score) list_score
   from student
   group by s_age
   ```

   - collect_list：将分组中的某列转为一个数组返回。

     在上述sql中，s_age有数据倾斜，但如果数据量大到一定的数量，会导致处理倾斜的Reduce任务产生内存溢出的异常。

     collect_list输出一个数组，中间结果会放到内存中，所以如果collect_list聚合太多数据，会导致内存溢出。

   有小伙伴说这是 group by 分组引起的数据倾斜，可以开启hive.groupby.skewindata参数来优化。我们接下来分析下：

   开启该配置会将作业拆解成两个作业，第一个作业会尽可能将Map的数据平均分配到Reduce阶段，并在这个阶段实现数据的预聚合，以减少第二个作业处理的数据量；第二个作业在第一个作业处理的数据基础上进行结果的聚合。

   hive.groupby.skewindata的核心作用在于生成的第一个作业能够有效减少数量。但是对于collect_list这类要求全量操作所有数据的中间结果的函数来说，明显起不到作用，反而因为引入新的作业增加了磁盘和网络I/O的负担，而导致性能变得更为低下。

   解决方案：

   ​	这类问题最直接的方式就是调整reduce所执行的内存大小。调整reduce的内存大小使用mapreduce.reduce.memory.mb这个配置。

**总结**

​	通过以上对数据倾斜场景的分析，不难看出，shuffle阶段最易发生数据倾斜，而且shuffle的过程也会产生大量的磁盘I/O、网络I/O以及压缩/解压缩、序列化/反序列化等。

[]: https://mp.weixin.qq.com/s?__biz=MzI0NjU2NDkzMQ==&amp;amp;mid=2247494450&amp;amp;idx=1&amp;amp;sn=389e3f6bbd89221bd46643ca6279d117&amp;amp;chksm=e9bffa1edec873080f725a6828fabfb02fa95850f61bdbf3310e55d6adee641451bfc753da1e&amp;amp;scene=21#wechat_redirect

