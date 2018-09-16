[TOC]

## 1 数据仓库和数据库的区别

**数据仓库：**

主要应用主要是在线分析处理(OLAP)，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。 

数据仓库是面向主题设计的 ，存储的是历史数据，有意引入冗余，集成先对稳定的，为数据分析而生的

常用的数据仓库有：Hive 、Greenplum、亚马逊



**数据库：**

要遵循ACID和范式要求，尽可能的减少冗余，保证完整，

数据库通常更关注**业务交易**处理（OLTP），是面向事务的设计; 

一般存储在线交易的数据，

常用的数据库：MySQL, Oracle, SqlServer等



**区别：**

1）数据库是面向事务的设计，数据仓库是面向主题设计的。 

2）数据库一般存储在线交易数据，数据仓库存储的一般是历史数据。

3）数据库设计是尽量避免冗余，数据仓库在设计是有意引入冗余。 

4）数据库是为捕获数据而设计，数据仓库是为分析数据而设计。 



## 2 数据仓库的物理(存储)模型

[数据仓库概念和物理模型](https://www.jianshu.com/p/e3aba572d945)

维度表：存储与业务相关的非核心信息的表。 一般记录流水，很大。 例如：销售清单，随着时间增长不断膨胀 

事实表：是反映业务核心的表，表中存储了与该业务相关的关键数据。一般固定、变动少、数据有限 

**1）星型模型** 

以事实表为中心，所有的维度表直接连接在事实表上，像星星一样。 

**特点：**数据组织直观，执行效率高。 

**2）雪花模型 （不太使用）**

雪花模型的维度表可以拥有其他维度表的

特点：这种模型不太容易理解，维护成本比较高，而且性能方面需要关联多层维表，性能也比星型模型要低 

**3）星座模型** 

星座模型是基于多张事实表的，而且共享维度信息。

通过构建一致性维度，来建设星座模型，也是很好的选择。比如同一主题的细节表和汇总表共享维度，不同主题的事实表，可以通过在维度上互相补充来生成可以共享的维度。 



## 3 数据分析

决策=数据+分析 

数据分析的框架：明确分析目标、数据收集、数据清理、数据分析、数据报告、执行与反馈 

数据分析与数据挖掘，前者偏向于业务分析，后者偏向于数据库算法，借助数据来指导决策 



## 4 Hive介绍

参考：[Hive体系架构](https://www.jianshu.com/p/fa5fe2694748)

1）Hive产生背景

MapReduce编程的不便性、DFS上的文件缺少Schema(表名，名称，ID等，为数据库对象的集合) 



2）Hive是什么

- 由Facebook开源，最初用于解决海量结构化的日志数据统计问题
- 构建在Hadoop之上的数据仓库，离线数据处理 
- Hive定义了一种类SQL查询语言：HQL 
- 底层支持多种不同的执行引擎 （MapReduce、Tez、Spark）



3）Hive 特点

使用简单、使用HQL来将数据映射成数据库表，使用Mysql来存储元数据、HDFS来提供存储、专为 OLAP 、不持支事务操作



4）缺点

**a：hive的HQL表达能力有限：**

迭代式算法无法表达、数据挖掘方面，比如kmeans

**b：hive的效率比较低：**

hive自动生成的mapreduce作业，通常情况下不够智能化；hive调优比较困难，粒度较粗；hive可控性差



## 5 Hive体系架构 

Hive是C/S模式 

![img](https://upload-images.jianshu.io/upload_images/5959612-73a17253df73e8a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/746/format/webp) 

客户端、服务器、驱动、元数据、sql执行引擎

**1）Client端： **

有JDBC/ODBC和Thrift Client，可远程访问Hive

可以通过shell脚本的方式访问，或者通过Thrift协议 



**2）Server端**

CLI、Thrift Server、HWI(Hive web Interface)、Driver、Metastore 

- 其中CLI、Thrift Server、HWI是暴露给Client访问的独立部署的Hive服务
- Driver、Metastore是Hive内部组件，Metastore还可以供第三方SQL on Hadoop框架使用
- beeine(Hive 0.11引入)，作为Hive JDBC Client访问HiveServer2，解决了**CLI并发访问**问题



**3）Driver**

a：进行HQL语句解析

b：从Metastore中获取表的信息

c：生成MR Job

d：再由Executor交给Hadoop运行

e：由Driver将结果返回给用户

> 输入了sql字符串，对sql字符串进行解析，转化程抽象语法树，再转化成逻辑计划，然后使用优化工具对逻辑计划进行优化，最终生成物理计划（序列化反序列化，UDF函数），交给Execution执行引擎，提交到MapReduce上执行（输入和输出可以是本地的也可以是HDFS/Hbase） 

具体的Driver过程：



> 解析器（Parser）：将查询字符串转化为解析树表达式
>
> 计划生成器（Planner）：表达式转化为MR Job
>
> **优化器：**只提供了基于规则的优化（列过滤 、行过滤 、谓词下推 、Join方式 ）
>
> 执行器（Execution Engine）：根据Job链（DAG），执行器会顺序执行其中所有的Job；如果Job链不存在依赖关系，采用并发方式进行Job的执行



4）元数据

存储和管理Hive的元数据，使用关系数据库来保存元数据信息。 



5）MR执行



## 6 Hive sql的执行流程

1）**执行查询：**hive界面如命令行或Web UI将查询发送到Driver(任何数据库驱动程序如JDBC、ODBC,等等)来执行。

2）**获得计划：**Driver根据查询编译器解析query语句,验证query语句的语法,查询计划或者查询条件。

3）**获取元数据：**编译器将元数据请求发送给Metastore(任何数据库)。

4）**接受元数据：**Metastore将元数据作为响应发送给编译器。

5）**发送：**编译器检查要求和重新发送Driver的计划。到这里,查询的解析和编译完成。

6）**执行计划：**Driver将执行计划发送到执行引擎。

7）**执行Job：**hadoop内部执行的是mapreduce。在执行引擎发送任务的同时，对hive的元数据进行相应操作。

8）**得到执行结果：**执行引擎接收数据节点(data node)的结果。

9）**返回结果：**执行引擎发送这些合成值到Driver。

10）**返回最终结果：**Driver将结果发送到hive接口。



## 7 数组组织格式|分区|桶

[Hive分区和桶](https://www.cnblogs.com/kxdblog/p/4316107.html)

1）表：每个表存储在HDFS上的一个目录下

2）分区：每个分区存储再表的子目录下 ，对应一个目录，Hive可以对数据按照某列或者某些列进行分区管理 

3）桶：某个分区根据某个列的hash值散列到不同的桶中，每个桶是一个文件 ，更为细粒度的数据范围划分 ，

Hive也是 针对某一列进行桶的组织。Hive采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。 



**把表（或者分区）组织成桶（Bucket）有两个理由：**

**1）获得更高的查询处理效率。**

桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。

**2）使取样（sampling）更高效**

在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。



## 8 内部表和外部表

1）内部表

数据是临时数据、外部的程序无法访问这些数据、数据会随着表的删除而删除 

内部表对数据拥有所有权，将内部表数据保存在hive.metastore.warehouse.dir目录下，删除内部表时，相应的数据也会被删除 ，表的创建过程和数据的加载时分别进行的



2）外部表

数据可以被外部程序访问、表被删除时，数据不会被删除 、 你不能基于已经存在的表再创建表 

Hive 对外部表的数据仅仅拥有使用权；外部表只有一个过程，加载数据和创建表同时完成 ，实际数据是存储在LOCATION后面指定的 HDFS 路径中，并不会移动到数据仓库目录中 



使用场景：

外部表：导入hdfs中的源数据 

内部表：存放Hive处理的中间表、结果表 

> 如： 每天将日志数据传入HDFS，一天一个目录；
>
> Hive基于流入的数据建立外部表，将每天HDFS上的原始日志映射到外部表的天分区中；
>
>  在外部表基础上做统计分析，使用内部表存储中间表、结果表，数据通过SELECT+INSERT进入内部表 





## 9 Hive支持的数据格式

文本文件、序列化文件(行)、parquet文件（列）、RCFile（列）、ORC（列）、Avro File（行） 

![image.png](https://upload-images.jianshu.io/upload_images/5959612-7909ebd6a9e9692d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





## 10 Hive的部署方式 

**1）内嵌模式：**

使用内嵌的Derby数据库作为存储元数据，Derby只能接受一个Hive会话的访问，不能用于生产； **hive服务、metastore服务、derby服务运行在同一个进程中。**

**2）本地模式：**

本地安装mysql，替代derby存储元数据，是一个多用户多客户端的模式，作为公司内部使用Hive；

**hive服务和metastore服务运行在同一个进程中，mysql数据库则是单独的进程，**可以同一台机器，也可以在远程机器上。

**3）远程模式（Remote）:** 

远程安装mysql 替代derby存储元数据；**Hive服务和metastore在不同的进程内，**也可能是不同的机器；

好处是：将Metastore分离出来，成为一个独立的Hive服务 可以将Mysql数据库层完全置于防火墙后，不再暴露数据库用户名和密码，避免认证信息的泄漏 



## 11 Hive的索引

Hive是支持索引的，但是很少被使用 

索引表不会自动rebuild，如果表有数据新增或删除，那么必须手动rebuild索引表数据 

索引表本身会非常大 Hive索引的使用过程比较繁琐 



Hive的两种索引： 

1）位图索引：普遍用于去重后值比较少的列    2）紧凑索引：存储每个值的HDFS块号 



## 12 数据倾斜定义和原因

绝大多数的任务执行的很快，只有极个别的执行很慢，或者原本可以正常执行的Spark作业，出现了OOM，究其原因是因为，大量的相同key落到了同一两个Reduce上



发生的原因：

1）key分布不均匀

2）业务本身的特性

3）建表时考虑不周全

4）一些sql语言本身具有数据倾斜（Join 、 group by、  count distinct）



## 13 数据倾斜的解决方法

**1）key分布不均匀**

key为null或异常值 ：对key进行打散，通过rand函数将为null的值分散到不同的值上； 对异常值赋一个随机值来分散key 

key为正常值：

a：设置reduce参数值，查过控制倾斜的阈值则将数据发送到其他的reduce上；

b：增加reduce的数量；

c：对业务需求进行细分



**2）group by产生的倾斜**

1：对Map 端部分聚合，相当于Combiner，设置hive.map.aggr=true 

2：设置hive.groupby.skewindata=true，控制生成两个MR Job，即就是生成两个MapReduce Job。

第一个Job进行预处理、部分聚合，使得结果是相同的Group By Key可以分发到不同的Reduce中；第二个MapReduce Job将预处理数据结果按照Group By Key分发到Reduce中，完成最终的聚合操作； 



**3）count distinct大量相同特殊值** 

count distinct时，将值为空的情况单独处理，如果是计算count distinct，可以不用处理，直接过滤，在最后结果中加1。如果还有其他计算，需要进行group by，可以先将值为空的记录单独处理，再和其他计算结果进行union(合并)。 



count()是全聚合操作，导致在map端的combine无法合并重复数据 ，即使设定了reduce task个数，set mapred.reduce.tasks=100；hive也只会启动一个reducer。这就造成了所有map端传来的数据都在一个tasks中执行，成为了性能瓶颈。 



**a：解决方式一（分治法）**

使用不同的reducer各自进行COUNT(DISTINCT)计算，充分发挥hadoop的优势，然后进行求和，间接达到了效果。需要注意的是多个tasks同时计算产生重复值的问题，所以分组需要使用到目标列的子串。 

```sql
SELECT SUM(tag) total
FROM
    (select substr(uid,1,4) tag, count(distinct substr(uid,5)) tmp_total
     from xxtable
     group by substr(uid,1,4)
     )t1
```



**b：解决方式二（随机分组法）**

核心是使用group by替代count(distinct)。

- 1层使用随机数作为分组依据，同时使用group by`保证去重`。
- 2层统计各分组下的统计数。
- 3层对分组结果求和。

```sql
--  第三层SELECT
SELECT
  SUM(s.mau_part) mau
FROM
(
  -- 第二层SELECT
  SELECT tag, COUNT(*) mau_part
  FROM
  (
    -- 第一层SELECT 为去重后的uuid打上标记，标记为：0-100之间的整数。
    SELECT uuid, CAST(RAND() * 100 AS BIGINT) tag  
    FROM detail_sdk_session
    WHERE partition_date >= '2016-01-01'  AND partition_date <= now AND uuid IS NOT NULL
    GROUP BY uuid   -- 通过GROUP BY，保证去重
   ) t
  GROUP BY tag
) s
;

链接：https://www.jianshu.com/p/b23717020e7b
```



**使用count(distinct ip)--->优化，去掉distinct ip 换成group by ip**

```sql
select count(distinct user_id) from dm_user where ds=20150701;

set mapred.reduce.tasks=50; 
select count(*) from 
(select user_id from dm_user where ds=20150701 group by user_id) t;
```



**4）Join优化**

1：将小表刷入内存中，可以设置刷入内存表的大小；将大表放在最后 （map join，减少shuffle的开销）

2：本地模式执行任务 

- 如果数据量 比较小，可以通过本地模式执行所有任务，即在执行的过程中通过设置为本地模式，因为本地模式下不会转换为mapreduce任务，而是将本地的数据文件格式输出 



**5）limit** 

开启limit优化，使用limit进行抽样查询，不需要全表扫描，缺点是有些需要的数据可能被忽略掉 



**6）并行执行** 

hive在执行查询的时候会将查询转化为**一个或多个Job链**，执行器会按照顺序执行这些Job；如果这些Job没有依赖关系，则可以采取并行方式进行执行。 



**7）启用严格模式** 



**8）Jvm重用** 

Hive Hql会转化成MapReduce，MR会将job任务转化为多个任务，每个任务都是一个新的JVM实例，重用这些实例，可以减少性能消耗。

> 默认情况下，每个task都是一个新的JVM实例，都需要开启和销毁。对于小文件，每个文件都会对应一个task，在一些情况下，**JVM开启和销毁的时间可能会比实际处理数据的时间消耗要长，**JVM重用是Hadoop调优参数的内容，其对Hive的性能具有非常大的影响，特别是对于小文件的场景，这类场景执行时间都很短 



**9）调整map和reduce的个数** 

有些只需要map，不需要reduce； hive是按照输入数据量的大小调整reducer个数，hive中可以配置一个reducer的数量大小，可以动态的调整； 



## 14 order by,sort by, distribute by, cluster by 

参考：[Hive之order by、sort by、distribute by和cluster by](https://www.jianshu.com/p/fb86b4ac4acf)

- **cluster by**字段含义：根据这个字段进行分区，需要注意设置reduce_num数量。

- **order by** 会对输入做全局排序，因此只有一个reducer，会导致当输入规模较大时，需要较长的计算时间。

- **sort by**不是全局排序，其在数据进入reducer前完成排序。因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1，则sort by只保证每个reducer的输出有序，不保证全局有序。

- **distribute by**(字段)根据指定的字段将数据分到不同的reducer，且分发算法是hash散列。distribute by必须要写在sort by之前。

- **Cluster by(字段)** 除了具有Distribute by的功能外，还会对该字段进行排序。

- 因此，如果分桶和sort字段是同一个时，此时，cluster by = distribute by + sort by

  

## 15 Hive避免进行MapReduce

参考：[Hive什么情况下可以避免进行MapReduce？](https://www.jianshu.com/p/97b48292ed8c)

hive 0.10.0为了执行效率考虑，简单的查询，就是只是select，不带count,sum,group by这样的，都不走map/reduce，直接读取hdfs文件进行filter过滤。 在hive-site.xml 设置参数

1、本地模式下，hive可以简单的读取目录路径下的数据，然后输出格式化后的数据到控制台，

2、查询语句中的过滤条件只是分区字段的情况下不会进行Mapreduce。 



## 16 Hive严格模式

参考：[Hive严格模式](https://www.jianshu.com/p/8a13b2750bf2)

1：分区查询时where中没有分区过滤条件，不允许扫描所有的分区； 

2：使用order by时必须使用limit限制，即可以防止reduce的额外执行时间； 

3：笛卡儿积，必须使用on字段，而不能使用where字句替代 

>注意on与where有什么区别，两个表连接时用on，在使用left  jion时，on和where条件的区别如下：
>
>1、on条件是在生成临时表时使用的条件，它不管on中的条件是否为真，都会返回左边表中的记录。
>
>2、where条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有left  join的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。





## 17 Hive基本操作

参考：[Hive基本操作](https://www.jianshu.com/p/2db188786e66)



## 18 Hive的join

**1）left semi join**

[hive中的LEFT SEMI JOIN](https://blog.csdn.net/bbbeoy/article/details/62236729)

**LEFT SEMI JOIN 是 IN/EXISTS 子查询的一种更高效的实现**。

Hive 当前**没有**实现 IN/EXISTS 子查询，所以你可以用 **LEFT SEMI JOIN 重写你的子查询语句**。LEFT SEMI JOIN 的限制是， JOIN 子句中右边的表只能在ON 子句中设置过滤条件，在 WHERE 子句、SELECT 子句或其他地方过滤都不行。 

```sql
SELECT a.key, a.value
FROM a
WHERE a.key in
   (SELECT b.key
    FROM B);
可以被重写为：
SELECT a.key, a.val
FROM a LEFT SEMI JOIN b on (a.key = b.key)
```







