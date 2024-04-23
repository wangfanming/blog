---
title: Hive性能问题排查
date: '2023/5/13 20:36:32'
categories:
  - 数据治理
  - 数据仓库
tags:
  - Hive

---

# Hive性能问题排查

## 背景

当我们发现一条SQL语句执行时间过长或者不合预期时，就需要我们对SQL进行优化分析了。怎么找到问题所在呢？

Hive SQL执行计划不像传统关系型数据库Oracle、SQL Server有真实的计划，可以看到各个阶段的处理数据、资源消耗、处理时间等量化数据。Hive的执行计划都是预测的，因此想要知道各个阶段的详细信息，只能查看Yarn提供的日志。日志链接在每个作业执行时给出，可以在控制台看到。

Hive执行计划目前可以查看到：

1. 查看执行计划的基本信息，即explain；
2. 查看执行计划的扩展信息，即explain extended；
3. 查看SQL数据输入依赖的信息，即explain dependency；
4. 查看SQL操作相关权限的信息，即explain authorization；
5. 查看SQL的向量化描述信息，即explain vectorization。

在SQL语句前加上explain关键词即可查看到执行计划的基本信息，基础的信息包括：

- 作业的依赖关系图，即STAGE DEPENDENCIES；
- 每个作业的详细信息，即STAGE PLANS。

<!--more-->

## Hive 性能调优手段

1.  **SQL语句优化**

   - union all

     ```sql
     insert into table stu partition(tp) 
     select s_age,max(s_birth) stat,'max' tp 
     from stu_ori
     group by s_age
     
     union all
     
     insert into table stu partition(tp) 
     select s_age,min(s_birth) stat,'min' tp 
     from stu_ori
     group by s_age;
     ```

     上述SQL目标很清晰，分别计算s_age的最大和最小值放在一张表内。但是存在这种写法会对同一张表相同字段进行两次分组，浪费性能。可以使用`from ... insert into ...`，这个语法将from前置，作用就是使用一张表，可以进行多次插入操作:

     ```sql
     --开启动态分区 
     set hive.exec.dynamic.partition=true; 
     set hive.exec.dynamic.partition.mode=nonstrict; 
     
     from stu_ori 
     
     insert into table stu partition(tp) 
     select s_age,max(s_birth) stat,'max' tp 
     group by s_age
     
     insert into table stu partition(tp) 
     select s_age,min(s_birth) stat,'min' tp 
     group by s_age;
     ```

   - distinct

     去重统计：

     ```sql
     select count(1) 
     from( 
       select s_age 
       from stu 
       group by s_age 
     ) b;	
     ```

     ```sql
     select count(distinct s_age) 
     from stu;
     ```

     - 上面进行去重的字段是年龄字段，但是年龄的枚举值是非常有限的，就算计算1岁到100岁之间的年龄，s_age的最大枚举值才100，如果转化成MapReduce来解释的话，在Map阶段，每个Map会对s_age去重。由于s_age枚举值有限，因而每个Map得到的s_age也有限，最终得到reduce的数据量也就是map数量s_age枚举值的个数。
     - distinct的命令会在内存中构建一个hashtable，查找去重的时间复杂度是O(1)；group by在不同版本间变动比较大，有的版本会用构建hashtable的形式去重，有的版本会通过排序的方式， 排序最优时间复杂度无法到O(1)。另外，第一种方式(group by)去重会转化为两个任务，会消耗更多的磁盘网络I/O资源。
     - 最新的Hive 3.0中新增了 count(distinct ) 优化，通过配置 hive.optimize.countdistinct，即使真的出现数据倾斜也可以自动优化，自动改变SQL执行的逻辑。
     - 第二种方式(distinct)比第一种方式(group by)代码简洁，表达的意思简单明了，如果没有特殊的问题，代码简洁就是优！
     - 数据格式优化

2. **存储组织格式**

   Hive提供了多种数据存储组织格式，不同格式对程序的运行效率也会有极大的影响。

   Hive提供的格式有TEXT、SequenceFile、RCFile、ORC和Parquet等。

   SequenceFile是一个二进制key/value对结构的平面文件，在早期的Hadoop平台上被广泛用于MapReduce输出/输出格式，以及作为数据存储格式。

   Parquet是一种列式数据存储格式，可以兼容多种计算引擎，如MapRedcue和Spark等，对多层嵌套的数据结构提供了良好的性能支持，是目前Hive生产环境中数据存储的主流选择之一。

   ORC优化是对RCFile的一种优化，它提供了一种高效的方式来存储Hive数据，同时也能够提高Hive的读取、写入和处理数据的性能，能够兼容多种计算引擎。事实上，在实际的生产环境中，ORC已经成为了Hive在数据存储上的主流选择之一。

   我们使用同样数据及SQL语句，只是数据存储格式不同，得到如下执行时长：

   ![数据存储格式对比](F:\study\blog\source\images\数据存储格式对比.png)

   > 注：CPU时间：表示运行程序所占用服务器CPU资源的时间。用户等待耗时：记录的是用户从提交作业到返回结果期间用户等待的所有时间。

   查询TextFile类型的数据表耗时33分钟， 查询ORC类型的表耗时1分52秒，时间得以极大缩短，可见不同的数据存储格式也能给HiveSQL性能带来极大的影响。

3. **小文件过多优化**

   在Hive执行过程中，每个小文件都会对应一个Map任务，而这个Map任务启动和初始化的时间远远大于逻辑处理时间，非常浪费资源。并且，同时可运行的Map数是有限的。

4. **并行执行优化**

   ​	Hive会将一个查询转化成一个或者多个阶段。这样的阶段可以是MapReduce阶段、抽样阶段、合并阶段、limit阶段，或者Hive执行过程中可能需要的其他阶段。默认情况下，Hive一次只会执行一个阶段。不过，某个特定的job可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个job的执行时间缩短。如果有更多的阶段可以并行执行，那么job可能就越快完成。

   通过设置参数hive.exec.parallel值为true，就可以开启并发执行。在共享集群中，需要注意下，如果job中并行阶段增多，那么集群利用率就会增加。

   ```shell
   set hive.exec.parallel=true; //打开任务并行执行
   set hive.exec.parallel.thread.number=16; //同一个sql允许最大并行度，默认为8。
   ```

   注：需要资源充足该配置才有意义。

5. JVM优化

   ​	JVM重用是Hadoop调优参数的内容，其对Hive的性能具有非常大的影响，特别是对于很难避免小文件的场景或task特别多的场景，这类场景大多数执行时间都很短。

   ​	Hadoop的默认配置通常是使用派生JVM来执行map和Reduce任务的。这时JVM的启动过程可能会造成相当大的开销，尤其是执行的job包含有成百上千task任务的情况。JVM重用可以使得JVM实例在同一个job中重新使用N次。N的值可以在Hadoop的mapred-site.xml文件中进行配置。通常在10-20之间，具体多少需要根据具体业务场景测试得出。

   ```xml
   <property>
     <name>mapreduce.job.jvm.numtasks</name>
     <value>10</value>
     <description>How many tasks to run per jvm. If set to -1, there is
     no limit. 
     </description>
   </property>
   ```

   我们也可以在hive中设置

   ```shell
   set  mapred.job.reuse.jvm.num.tasks=10; //这个设置来设置我们的jvm重用
   ```

   **这个功能的缺点是，开启JVM重用将一直占用使用到的task插槽，以便进行重用，直到任务完成后才能释放。如果某个“不平衡的”job中有某几个reduce task执行的时间要比其他Reduce task消耗的时间多的多的话，那么保留的插槽就会一直空闲着却无法被其他的job使用，直到所有的task都结束了才会释放**。

6. **推测执行优化**

   ​	在分布式集群环境下，因为程序Bug（包括Hadoop本身的bug），负载不均衡或者资源分布不均等原因，会造成同一个作业的多个任务之间运行速度不一致，有些任务的运行速度可能明显慢于其他任务（比如一个作业的某个任务进度只有50%，而其他所有任务已经运行完毕），则这些任务会拖慢作业的整体执行进度。为了避免这种情况发生，Hadoop采用了推测执行（Speculative Execution）机制，它根据一定的法则推测出“拖后腿”的任务，并为这样的任务启动一个备份任务，让该任务与原始任务同时处理同一份数据，并最终选用最先成功运行完成任务的计算结果作为最终结果。

   设置开启推测执行参数：Hadoop的mapred-site.xml文件中进行配置：

   ```javascript
   <property>
     <name>mapreduce.map.speculative</name>
     <value>true</value>
     <description>If true, then multiple instances of some map tasks 
                  may be executed in parallel.</description>
   </property>
   
   <property>
     <name>mapreduce.reduce.speculative</name>
     <value>true</value>
     <description>If true, then multiple instances of some reduce tasks 
                  may be executed in parallel.</description>
   </property>
   ```

   hive本身也提供了配置项来控制reduce-side的推测执行:

   ```shell
   set hive.mapred.reduce.tasks.speculative.execution=true
   ```

   ​	如果用户对于运行时的偏差非常敏感的话，那么可以将这些功能关闭掉。如果用户因为输入数据量很大而需要执行长时间的map或者Reduce task的话，那么启动推测执行造成的浪费是非常巨大大。

代码优化原则：

- 理透需求原则，这是优化的根本；
- 把握数据全链路原则，这是优化的脉络；
- 坚持代码的简洁原则，这让优化更加简单；
- 没有瓶颈时谈论优化，这是自寻烦恼。



[]: https://cloud.tencent.com/developer/article/1893127

