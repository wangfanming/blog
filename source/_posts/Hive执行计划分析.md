---
title: Hive执行计划分析
date: '2023/5/9 16:16:32'
categories:
  - 数据治理
  - 数据仓库
tags:
  - Hive
---

# Hive执行计划分析

## 背景

Apache Hive作为构建数据仓库最常用的技术，借助SQL语法的简洁性，使得数据处理过程变得更易上手。但是，有一利必有一弊。易用性的背后是Hive技术实现了非常多的手段对各种场景的预判。业务场景千千万，再强的技术也不能一刀切，所以需要我们深入了解该技术是怎么执行我们的任务的，以及怎么根据现有任务的运行情况，通过参数设置、SQL调优、设计方案调整等手段来“多快好省”地解决我们的问题。

<!--more-->

## Hive执行计划

Hive执行计划描述了SQL实际执行时是怎么被转换成计算引擎的执行逻辑的。了解并掌握了这些执行逻辑，有助于我们更好地分析程序的阻塞点，从而能有的放矢地解决性能问题。

- ### 查看SQL执行计划

  - explain:查看执行计划的基本信息
  - explain dependency：可以查看到执行计划的输入信息，包含了各种属性。
  - explain authorization：查看SQL操作相关权限的信息；
  - explain vectorization：查看SQL的向量化描述信息，显示为什么未对Map和Reduce进行矢量化。从 Hive 2.3.0 开始支持；
  - explain analyze：用实际的行数注释计划。从 Hive 2.2.0 开始支持；

  1. **explain的用法**

     语法：`explain query;`

     例如：

     ​	在Hive cli（Hive2.3.7）中输入：

     ```SQL
     explain select sum(id) from test1;
     ```

     输出：

     ```
     STAGE DEPENDENCIES:
       Stage-1 is a root stage
       Stage-0 depends on stages: Stage-1
     
     STAGE PLANS:
       Stage: Stage-1
         Map Reduce
           Map Operator Tree:
               TableScan
                 alias: test1
                 Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                 Select Operator
                   expressions: id (type: int)
                   outputColumnNames: id
                   Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                   Group By Operator
                     aggregations: sum(id)
                     mode: hash
                     outputColumnNames: _col0
                     Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
                     Reduce Output Operator
                       sort order:
                       Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
                       value expressions: _col0 (type: bigint)
           Reduce Operator Tree:
             Group By Operator
               aggregations: sum(VALUE._col0)
               mode: mergepartial
               outputColumnNames: _col0
               Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
               File Output Operator
                 compressed: false
                 Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
                 table:
                     input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                     output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                     serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
     
       Stage: Stage-0
         Fetch Operator
           limit: -1
           Processor Tree:
             ListSink
     ```

     上述可以看出，一个Hive查询会被划分为多个stage组成的DAG。

     stage的类型包括：

     - MapReduceStage
     - 负责元数据的stage
     - 负责文件系统操作的stage

     上述explain包含的两大部分：

     先看第一部分 stage dependencies，描述了stage之间的依赖关系。

     - stage dependencies：各个stage之间的依赖性
     - stage plan：各个stage的执行计划

     再看第二部分 stage plan，里面有一个 Map Reduce，一个MR的执行计划分为两个部分：

     - Map Operator Tree：MAP端的执行计划树
     - Reduce Operator Tree：Reduce端的执行计划树

     这两个执行计划树里面包含这条sql语句的 operator：

     1. TableScan：表扫描操作，map端第一个操作肯定是加载表，所以就是表扫描操作，常见的属性：

        - alias：表名称

        - Statistics：表统计信息，包含表中数据条数，数据大小等

     2. Select Operator：选取操作，常见的属性 ：

        - expressions：需要的字段名称及字段类型

        - outputColumnNames：输出的列名称

        - Statistics：表统计信息，包含表中数据条数，数据大小等

     3. Group By Operator：分组聚合操作，常见的属性：

        - aggregations：显示聚合函数信息

        - mode：聚合模式，值有 hash：随机聚合，就是hash partition；partial：局部聚合；final：最终聚合

        - keys：分组的字段，如果没有分组，则没有此字段

        - outputColumnNames：聚合之后输出列名

        - Statistics：表统计信息，包含分组聚合之后的数据条数，数据大小等

     4. Reduce Output Operator：输出到reduce操作，常见属性：

        - sort order：值为空 不排序；值为 + 正序排序，值为 - 倒序排序；值为 +- 排序的列为两列，第一列为正序，第二列为倒序
        - Statistics：表统计信息，包含分组聚合之后的数据条数，数据大小等

     5. Filter Operator：过滤操作，常见的属性：

        - predicate：过滤条件，如sql语句中的where id>=1，则此处显示(id >= 1)

     6. Map Join Operator：join 操作，常见的属性：

        - condition map：join方式 ，如Inner Join 0 to 1 Left Outer Join0 to 2

        - keys: join 的条件字段

        - outputColumnNames：join 完成之后输出的字段

        - Statistics：join 完成之后生成的数据条数，大小等

     7. File Output Operator：文件输出操作，常见的属性

        - compressed：是否压缩

        - table：表的信息，包含输入输出文件格式化方式，序列化方式等

     8. Fetch Operator 客户端获取数据操作，常见的属性：

        - limit，值为 -1 表示不限制条数，其他值为限制的条数

  2. **explain的使用场景**

     1. join 语句会过滤 null 的值吗？

        现在，我们在hive cli 输入以下查询计划语句

        ```sql
        select a.id,b.user_name from test1 a join test2 b on a.id=b.id;
        ```

        问：上面这条 join 语句会过滤 id 为 null 的值吗

        执行下面语句：

        ```sql
        explain select a.id,b.user_name from test1 a join test2 b on a.id=b.id;
        ```

        我们来看结果 (为了适应页面展示，仅截取了部分输出信息)：

        ```sql
        TableScan
         alias: a
         Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
         Filter Operator
            predicate: id is not null (type: boolean)
            Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
            Select Operator
                expressions: id (type: int)
                outputColumnNames: _col0
                Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                HashTable Sink Operator
                   keys:
                     0 _col0 (type: int)
                     1 _col0 (type: int)
         ...
        ```

        从上述结果可以看到 predicate: id is not null 这样一行，说明 join 时会自动过滤掉关联字段为 null 值的情况，但 left join 或 full join 是不会自动过滤null值的.

     2. group by 分组语句会进行排序吗？

        ```sql
        select id,max(user_name) from test1 group by id;
        ```

        explain的 结果 (为了适应页面展示，仅截取了部分输出信息)

        ```sql
         TableScan
            alias: test1
            Statistics: Num rows: 9 Data size: 108 Basic stats: COMPLETE Column stats: NONE
            Select Operator
                expressions: id (type: int), user_name (type: string)
                outputColumnNames: id, user_name
                Statistics: Num rows: 9 Data size: 108 Basic stats: COMPLETE Column stats: NONE
                Group By Operator
                   aggregations: max(user_name)
                   keys: id (type: int)
                   mode: hash
                   outputColumnNames: _col0, _col1
                   Statistics: Num rows: 9 Data size: 108 Basic stats: COMPLETE Column stats: NONE
                   Reduce Output Operator
                     key expressions: _col0 (type: int)
                     sort order: +
                     Map-reduce partition columns: _col0 (type: int)
                     Statistics: Num rows: 9 Data size: 108 Basic stats: COMPLETE Column stats: NONE
                     value expressions: _col1 (type: string)
         ...
        ```

        我们看 Group By Operator，里面有 keys: id (type: int) 说明按照 id 进行分组的，再往下看还有 sort order: + ，说明是按照 id 字段进行正序排序的。

     3. 哪条sql执行效率高呢？

        ```sql
        SELECT
         a.id,
         b.user_name
        FROM
         test1 a
        JOIN test2 b ON a.id = b.id
        WHERE
         a.id > 2;
        ```

        ```sql
        SELECT
         a.id,
         b.user_name
        FROM
         (SELECT * FROM test1 WHERE id > 2) a
        JOIN test2 b ON a.id = b.id;
        ```

        第一条sql explain:

        ```sql
        hive (default)> explain select a.id,b.user_name from test1 a join test2 b on a.id=b.id where a.id >2;
        OK
        Explain
        STAGE DEPENDENCIES:
          Stage-4 is a root stage
          Stage-3 depends on stages: Stage-4
          Stage-0 depends on stages: Stage-3
        
        STAGE PLANS:
          Stage: Stage-4
            Map Reduce Local Work
              Alias -> Map Local Tables:
                $hdt$_0:a
                  Fetch Operator
                    limit: -1
              Alias -> Map Local Operator Tree:
                $hdt$_0:a
                  TableScan
                    alias: a
                    Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                    Filter Operator
                      predicate: (id > 2) (type: boolean)
                      Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                      Select Operator
                        expressions: id (type: int)
                        outputColumnNames: _col0
                        Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                        HashTable Sink Operator
                          keys:
                            0 _col0 (type: int)
                            1 _col0 (type: int)
        
          Stage: Stage-3
            Map Reduce
              Map Operator Tree:
                  TableScan
                    alias: b
                    Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                    Filter Operator
                      predicate: (id > 2) (type: boolean)
                      Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                      Select Operator
                        expressions: id (type: int), user_name (type: string)
                        outputColumnNames: _col0, _col1
                        Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                        Map Join Operator
                          condition map:
                               Inner Join 0 to 1
                          keys:
                            0 _col0 (type: int)
                            1 _col0 (type: int)
                          outputColumnNames: _col0, _col2
                          Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                          Select Operator
                            expressions: _col0 (type: int), _col2 (type: string)
                            outputColumnNames: _col0, _col1
                            Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                            File Output Operator
                              compressed: false
                              Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                              table:
                                  input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                                  output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                                  serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
              Local Work:
                Map Reduce Local Work
        
          Stage: Stage-0
            Fetch Operator
              limit: -1
              Processor Tree:
                ListSink
        ```

        第二条sql explain:

        ```sql
        hive (default)> explain select a.id,b.user_name from(select * from  test1 where id>2 ) a join test2 b on a.id=b.id;
        OK
        Explain
        STAGE DEPENDENCIES:
          Stage-4 is a root stage
          Stage-3 depends on stages: Stage-4
          Stage-0 depends on stages: Stage-3
        
        STAGE PLANS:
          Stage: Stage-4
            Map Reduce Local Work
              Alias -> Map Local Tables:
                $hdt$_0:test1
                  Fetch Operator
                    limit: -1
              Alias -> Map Local Operator Tree:
                $hdt$_0:test1
                  TableScan
                    alias: test1
                    Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                    Filter Operator
                      predicate: (id > 2) (type: boolean)
                      Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                      Select Operator
                        expressions: id (type: int)
                        outputColumnNames: _col0
                        Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                        HashTable Sink Operator
                          keys:
                            0 _col0 (type: int)
                            1 _col0 (type: int)
        
          Stage: Stage-3
            Map Reduce
              Map Operator Tree:
                  TableScan
                    alias: b
                    Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
                    Filter Operator
                      predicate: (id > 2) (type: boolean)
                      Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                      Select Operator
                        expressions: id (type: int), user_name (type: string)
                        outputColumnNames: _col0, _col1
                        Statistics: Num rows: 2 Data size: 25 Basic stats: COMPLETE Column stats: NONE
                        Map Join Operator
                          condition map:
                               Inner Join 0 to 1
                          keys:
                            0 _col0 (type: int)
                            1 _col0 (type: int)
                          outputColumnNames: _col0, _col2
                          Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                          Select Operator
                            expressions: _col0 (type: int), _col2 (type: string)
                            outputColumnNames: _col0, _col1
                            Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                            File Output Operator
                              compressed: false
                              Statistics: Num rows: 2 Data size: 27 Basic stats: COMPLETE Column stats: NONE
                              table:
                                  input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                                  output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                                  serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
              Local Work:
                Map Reduce Local Work
        
          Stage: Stage-0
            Fetch Operator
              limit: -1
              Processor Tree:
                ListSink
        ```

        通过执行计划可以看出，两条SQL在执行时都时先进行 where 条件过滤，在进行 join 条件关联。因此执行效率是一样的。

  3. **explain dependency的用法**

     explain dependency用于描述一段SQL需要的数据来源，输出是一个json格式的数据，里面包含以下两个部分的内容：

     - input_partitions：描述一段SQL依赖的数据来源表分区，里面存储的是分区名的列表，如果整段SQL包含的所有表都是非分区表，则显示为空。
     - input_tables：描述一段SQL依赖的数据来源表，里面存储的是Hive表名的列表。

     使用explain dependency查看SQL查询非分区普通表，在 hive cli 中输入以下命令：

     ```
     explain dependency select s_age,count(1) num from student_orc;
     ```

     得到结果：

     ```
     {"input_partitions":[],"input_tables":[{"tablename":"default@student_tb _orc","tabletype":"MANAGED_TABLE"}]}
     ```

     使用explain dependency查看SQL查询分区表，在 hive cli 中输入以下命令：

     ```
     explain dependency select s_age,count(1) num from student_orc_partition;
     ```

     得到结果：

     ```
     {"input_partitions":[{"partitionName":"default@student_orc_partition@ part=0"}, 
     {"partitionName":"default@student_orc_partition@part=1"}, 
     {"partitionName":"default@student_orc_partition@part=2"}, 
     {"partitionName":"default@student_orc_partition@part=3"},
     {"partitionName":"default@student_orc_partition@part=4"}, 
     {"partitionName":"default@student_orc_partition@part=5"},
     {"partitionName":"default@student_orc_partition@part=6"},
     {"partitionName":"default@student_orc_partition@part=7"},
     {"partitionName":"default@student_orc_partition@part=8"},
     {"partitionName":"default@student_orc_partition@part=9"}], 
     "input_tables":[{"tablename":"default@student_orc_partition", "tabletype":"MANAGED_TABLE"}]
     ```

     explain dependency的使用场景有两个：

     - 场景一：快速排除。快速排除因为读取不到相应分区的数据而导致任务数据输出异常。例如，在一个以天分区的任务中，上游任务因为生产过程不可控因素出现异常或者空跑，导致下游任务引发异常。通过这种方式，可以快速查看SQL读取的分区是否出现异常。
     - 场景二：理清表的输入，帮助理解程序的运行，特别是有助于理解有多重子查询，多表连接的依赖输入

     案例：

     1. 识别看似等价的代码

        ```sql
        select * from a inner join b on a.no=b.no and a.f>1 and a.f<3;
        ```

        与

        ```sql
        select * from a inner join b on a.no=b.no where a.f>1 and a.f<3;
        ```

        是否等价？

        代码1：

        ```sql
        select 
        a.s_no 
        from student_orc_partition a 
        inner join 
        student_orc_partition_only b 
        on a.s_no=b.s_no and a.part=b.part and a.part>=1 and a.part<=2;
        ```

        代码2：

        ```sql
        select 
        a.s_no 
        from student_orc_partition a 
        inner join 
        student_orc_partition_only b 
        on a.s_no=b.s_no and a.part=b.part 
        where a.part>=1 and a.part<=2;
        ```

        我们看下上述两段代码explain dependency的输出结果：

        代码1的explain dependency结果：

        ```sql
        {"input_partitions": 
        [{"partitionName":"default@student_orc_partition@part=0"}, 
        {"partitionName":"default@student_orc_partition@part=1"}, 
        {"partitionName":"default@student_orc_partition@part=2"},
        {"partitionName":"default@student_orc_partition_only@part=1"}, 
        {"partitionName":"default@student_orc_partition_only@part=2"}], 
        "input_tables": [{"tablename":"default@student_orc_partition","tabletype":"MANAGED_TABLE"}, {"tablename":"default@student_orc_partition_only","tabletype":"MANAGED_TABLE"}]}
        ```

        代码2的explain dependency结果：

        ```sql
        {"input_partitions": 
        [{"partitionName":"default@student_orc_partition@part=1"}, 
        {"partitionName" : "default@student_orc_partition@part=2"},
        {"partitionName" :"default@student_orc_partition_only@part=1"},
        {"partitionName":"default@student_orc_partition_only@part=2"}], 
        "input_tables": [{"tablename":"default@student_orc_partition","tabletype":"MANAGED_TABLE"}, {"tablename":"default@student_orc_partition_only","tabletype":"MANAGED_TABLE"}]}
        ```

        通过上面的输出结果可以看到，其实上述的两个SQL并不等价，代码1在内连接（inner join）中的连接条件（on）中加入非等值的过滤条件后，并没有将内连接的左右两个表按照过滤条件进行过滤，内连接在执行时会多读取part=0的分区数据。而在代码2中，会过滤掉不符合条件的分区。

     2. 识别SQL读取数据范围的差别

        代码1：

        ```sql
        explain dependency
        select
        a.s_no 
        from student_orc_partition a 
        left join 
        student_orc_partition_only b 
        on a.s_no=b.s_no and a.part=b.part and b.part>=1 and b.part<=2;
        ```

        代码2：

        ```sql
        explain dependency 
        select 
        a.s_no 
        from student_orc_partition a 
        left join 
        student_orc_partition_only b 
        on a.s_no=b.s_no and a.part=b.part and a.part>=1 and a.part<=2;
        ```

        以上两个代码的数据读取范围是一样的吗？答案是不一样，我们通过explain dependency来看下：

        代码1的explain dependency结果：

        ```sql
        {"input_partitions": 
        [{"partitionName": "default@student_orc_partition@part=0"}, 
        {"partitionName":"default@student_orc_partition@part=1"}, …中间省略7个分区
        {"partitionName":"default@student_orc_partition@part=9"}, 
        {"partitionName":"default@student_orc_partition_only@part=1"}, 
        {"partitionName":"default@student_orc_partition_only@part=2"}], 
        "input_tables": [{"tablename":"default@student_orc_partition","tabletype":"MANAGED_TABLE"}, {"tablename":"default@student_orc_partition_only","tabletype":"MANAGED_TABLE"}]}
        ```

        代码2的explain dependency结果：

        ```sql
        {"input_partitions": 
        [{"partitionName":"default@student_orc_partition@part=0"}, 
        {"partitionName":"default@student_orc_partition@part=1"}, …中间省略7个分区 
        {"partitionName":"default@student_orc_partition@part=9"}, 
        {"partitionName":"default@student_orc_partition_only@part=0"}, 
        {"partitionName":"default@student_orc_partition_only@part=1"}, …中间省略7个分区 
        {"partitionName":"default@student_orc_partition_only@part=9"}],
        "input_tables": [{"tablename":"default@student_orc_partition","tabletype":"MANAGED_TABLE"}, {"tablename":"default@student_orc_partition_only","tabletype":"MANAGED_TABLE"}]}
        ```

        可以看到，对左外连接在连接条件中加入非等值过滤的条件，如果过滤条件是作用于右表（b表）有起到过滤的效果，则右表只要扫描两个分区即可，但是左表（a表）会进行全表扫描。如果过滤条件是针对左表，则完全没有起到过滤的作用，那么两个表将进行全表扫描。这时的情况就如同全外连接一样都需要对两个数据进行全表扫描。

        在使用过程中，容易认为代码片段2可以像代码片段1一样进行数据过滤，通过查看explain dependency的输出结果，可以知道不是如此。

  4. **explain authorization 的用法**

     通过explain authorization可以知道当前SQL访问的数据来源（INPUTS） 和数据输出（OUTPUTS），以及当前Hive的访问用户 （CURRENT_USER）和操作（OPERATION）。

     在 hive cli 中输入以下命令：

     ```sql
     explain authorization 
     select variance(s_score) from student_tb_orc;
     ```

     结果如下：

     ```sql
     INPUTS: 
       default@student_tb_orc 
     OUTPUTS: 
       hdfs://node01:8020/tmp/hive/hdfs/cbf182a5-8258-4157-9194- 90f1475a3ed5/-mr-10000 
     CURRENT_USER: 
       hdfs 
     OPERATION: 
       QUERY 
     AUTHORIZATION_FAILURES: 
       No privilege 'Select' found for inputs { database:default, table:student_ tb_orc, columnName:s_score}
     ```

     从上面的信息可知：

     上面案例的数据来源是defalut数据库中的 student_tb_orc表；

     数据的输出路径是hdfs://node01:8020/tmp/hive/hdfs/cbf182a5-8258-4157-9194-90f1475a3ed5/-mr-10000；

     当前的操作用户是hdfs，操作是查询；

     观察上面的信息我们还会看到AUTHORIZATION_FAILURES信息，提示对当前的输入没有查询权限，但如果运行上面的SQL的话也能够正常运行。为什么会出现这种情况？Hive在默认不配置权限管理的情况下不进行权限验证，所有的用户在Hive里面都是超级管理员，即使不对特定的用户进行赋权，也能够正常查询。