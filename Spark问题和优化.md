## ----------------RDD-----------------

## 1  RDD介绍

Spark的编程抽象模型：RDD和共享变量（支持并行计算的广播变量和累加器）



**共享变量**

在并行计算中使用共享变量

 1）广播变量 可以在内存的所有节点中被访问，用于缓存变量（只读），减少通信的代价 SparkContext.broadcast(v) 

2）累加器 只能用来做加法的变量，如计数和求和 SparkContext.arrumulator(v) 



**RDD：**是弹性分布式数据集，支持多种来源（本地文件，HDFS文件，HBase，Amazon S3等），具有容错机制，可以被缓存，支持并行操作（从各种分布式文件系统创建 ，从支持Hadoop输入格式数据源创建 ）



**RDD特征或数据结构：**分区，依赖，函数，优先位置，分区策略

1）分区：能够将数据切分，切分后的数据能够进行并行计算 

2）函数：计算每个分片，得到可遍历的结果 

3）依赖：通过依赖关系描述血统 

4）优先位置：（可选）对每个分片可优先计算位置 

5）分区策略：（可选）描述分区模式和数据存放的位置，以键值对形式根据哈希值进行分区，如根据key来决定分配位置 

>分区（Partition）相关接口 
>
>依赖（Dependency）相关接口 
>
>计算（Computing）相关接口 
>
>分区器（Partitioner）相关接口（可选） 
>
>首选位置（Prefered Location）相关接口（可选） 
>
>持久化（Persistence）
>
>检查点（Checkpoint）相关接口 



**RDD的缺点：**

不支持细粒度的计算，不支持增量迭代运算（计算时只计算一部分数据）。

>spark声称支持迭代计算，是因为中间数据在内存中，但是RDD是只读的，而一般来说迭代计算都会需要进行增量的修改，这种情况spark该如何处理？ 



## 2 RDD的窄依赖和宽依赖

[spark面试问题](https://blog.csdn.net/wyfly69/article/details/79950128)

宅依赖：父分区最多只能被一个子分区使用，即就是：1：1  n：1

宽依赖：父分区被多个子分区使用 ，即就是：1：n

![image.png](https://upload-images.jianshu.io/upload_images/5959612-0228b3e2653b558e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**窄依赖RDD：**

可以通过相同的键进行联合分区，整个操作都可以在一个集群节点上进行，以流水线（pipeline）的方式计算所有父分区，不会造成网络之间的数据shuffle，不同节点之间可以并行计算 （可以进行并行计算，不会产生shuffle）

**涉及有窄依赖的有：**map，filter，union，join(父RDD是hash-partition)，mapPartition，mapValues



**宽依赖：**

宽依赖的RDD之间会涉及到shuffle，宽依赖首先要计算好所有的父分区数据，然后在节点之间进行shuffle

**涉及的宽依赖有：**groupByKey，reduceByKey，join(父分区不是hash-partition)，partionBy



窄依赖可以进行有效的容错和失效节点的恢复，重新计算丢失的父分区，不同节点之间可以进行并行计算；

而宽依赖，单个子分区失效，恢复时可能导致要重新计算所有的父分区

注意：Shuffle执行固化操作，以及采用persist缓存策略，可以在固化点、缓存点重新计算 (设置固化点)



## 3 RDD操作

**1）转化**

在RDD之间指定处理的相互依赖关系，是数据集的逻辑操作，并没有正在执行（转化都是惰性操作，遇到action才执行）

常用的转化操作：

a：基础转化操作：map、filter、flatMap、mapPartition、union、dist

b：键值转化操作：groupByKey、reduceByKey、sortByKey、join、cogroup

>join：<k，v>  join <k, w>   --->  (k, <v, w>)



**2）执行**

将前一个Action和提交的Action之间的所有Transformation组成的Job进行计算，并根据Action将作业切分成多个Job，指定转化操作的输出结果 



常用执行操作：reduce、collect、count、first、take、countByKey、foreach

存储执行操作：saveAsTextFile、saveAsSequenceFile、saveAsObjectFile



## 4 RDD的基本算子

map：是将函数用于RDD中的每个元素，将返回值构成新的RDD。 （transformation算子）

mapPartition：用于遍历操作RDD中的每一个分区，返回生成一个新的RDD（transformation算子）。 

flatMap：将函数应用于RDD中的每一个元素，并将返回的迭代器中的所有内容构成新的RDD，常用于切分单词； 

>参考：[Spark之中map与flatMap的区别](https://blog.csdn.net/SparkOrHadoop/article/details/78226756)
>
>输入：[This is a pen]
>
>map：对集合中每个元素进行操作。  [<This,1> ，<is,2>，<a,1>，<pen,1>]对每一条输入进行指定的操作，然后为每一条输入返回一个对象 
>
>flatMap：对集合中每个元素进行操作然后再扁平化。  [This，is，a，pen]最后将所有对象合并为一个对象 



fileter：根据条件过滤，返回新的RDD（transformation算子）。 

union()：生成一个包含两个RDD中的所有元素的RDD （transformation算子）。 



collect：把所有的数据汇总到Driver的JVM内存中，对Driver的压力很大，尽量不要使用，可以保存为文本在查看



foreach：用于遍历RDD,将函数应用于每一个元素，无返回值 (action算子)。 

foreachPartition: 用于遍历操作RDD中的每一个分区。无返回值(action算子)。 



**RDD集合中对键值对的操作：**

例如：{（1,2），（3,4），（3,5,）}

reduceByKey ：合并具有相同键的值    rdd.reduceByKey((x,y) => x+y)   ===>  {（1,2）, （3,9）}

groupByKey：对具有相同键的值进行分组，   rdd.groupByKey( ) = {（1，[2] ），（3，[4,5] ）} 

>在大的数据集上，reduceByKey()的效果比groupByKey()的效果更好一些。因为reduceByKey()会在shuffle之前对数据进行合并。 



**RDD的join操作：**

join()：对两个RDD进行内连接 ，返回集合配对成功的，过滤关联不上的数据

leftOuterJoin：对两个RDD进行左连接，返回结果以左面的RDD为主，关联不上的记录为空。

rightOuterJoin：对两个RDD进行右连接，返回结果以右边的RDD为主，关联不上的记录为空。

```scala
//建立一个基本的键值对RDD，包含ID和名称，其中ID为1、2、3、4  
val rdd1 = sc.makeRDD(Array(("1","Spark"),("2","Hadoop"),("3","Scala"),("4","Java")),2)  
//建立一个行业薪水的键值对RDD，包含ID和薪水，其中ID为1、2、3、5  
val rdd2 = sc.makeRDD(Array(("1","30K"),("2","15K"),("3","25K"),("5","10K")),2)  

println("//下面做Join操作，预期要得到（1,×）、（2,×）、（3,×）")  
val joinRDD=rdd1.join(rdd2).collect.foreach(println)  

println("//下面做leftOutJoin操作，预期要得到（1,×）、（2,×）、（3,×）、(4,×）")  
val leftJoinRDD=rdd1.leftOuterJoin(rdd2).collect.foreach(println)  
println("//下面做rightOutJoin操作，预期要得到（1,×）、（2,×）、（3,×）、(5,×）")  
val rightJoinRDD=rdd1.rightOuterJoin(rdd2).collect.foreach(println)  

//下面做Join操作，预期要得到（1,×）、（2,×）、（3,×）
(2,(Hadoop,15K))
(3,(Scala,25K))
(1,(Spark,30K))
//下面做leftOutJoin操作，预期要得到（1,×）、（2,×）、（3,×）、(4,×）
(4,(Java,None))
(2,(Hadoop,Some(15K)))
(3,(Scala,Some(25K)))
(1,(Spark,Some(30K)))
//下面做rightOutJoin操作，预期要得到（1,×）、（2,×）、（3,×）、(5,×）
(2,(Some(Hadoop),15K))
(5,(None,10K))
(3,(Some(Scala),25K))
(1,(Some(Spark),30K))
```



## 5 RDD故障恢复|持久化|存储等级

**1）RDD故障恢复**
跨宽依赖的再执行能够涉及多个父RDD，从而引发全部的再执行，为了避免，Spark会保持Map阶段中间数据输出持久到内存/磁盘（持久化中间的RDD）

Spark容错：
基于RDD的容错
数据检查点和记录日志，用于持久化中间RDD；通过比较恢复延迟和检查点开销进行权衡，spark会自动化地选择相应的策略进行故障恢复

**2）RDD持久化**
是指：在不同转换操作中，将过程数据缓存到内存中，实现快速重用/故障快速恢复。
设置不同级别的持久化，从而允许持久化数据集在硬盘或内存作为序列化的Java对象（节省空间），进行跨节点复制。

主动持久化：persist() unpersist()
自动持久化：Spark自动地保存一些Shuffle操作

**3）选择存储等级**
MEMORY_ONLY：使用未序列化的Java对象格式，将数据保存在内存中（JVM）。如果内存不够存放所有的数据，则数据可能就不会进行持久化，是默认的持久化策略，效率最高

MEMORY_AND_DISK：使用未序列化的Java对象格式，优先尝试将数据保存在内存中。如果内存不够存放所有的数据，会将数据写入磁盘文件中。不会立刻输出到磁盘(先)，适用于计算量特别大，或者过滤了大量的数据。

MEMORY_ONLY_SER:将RDD作为序列化的Java对象进行存储，节省空间，但读取时占用比较占用CPU

DISK_ONLY：只将RDD分区存储在硬盘上



## 6 RDD的MapReduce操作

lines.flatMap(_.split(" ")).map(\_,1).reduceByKey(\_+\+);



## 7 map和mapPartition的区别？ 

map()：是将函数用于RDD中的每个元素，将返回值构成新的RDD。 



rdd的mapPartitions是map的一个变种，它们都可进行分区的并行处理。 

两者的主要区别是调用的粒度不一样：map的输入变换函数是应用于RDD中每个元素，而mapPartitions的输入函数是应用于每个分区。 



flatmap()：是将函数应用于RDD中的每个元素，将返回的迭代器的所有内容构成新的RDD，这样就得到了一个由各列表中的元素组成的RDD,而不是一个列表组成的RDD。 即就是：flatMap = flat + map 



## 8 spark中cache和persist的区别

1）cache：缓存数据，默认是缓存在内存中，其本质还是调用persist

2）persist:缓存数据，有丰富的数据缓存策略。数据可以保存在内存也可以保存在磁盘中，使用的时候指定对应的缓存级别就可以了。



参考：[spark面试问题收集](https://blog.csdn.net/wyfly69/article/details/79950128)



## 9 spark 如何防止内存溢出 

1）driver端的内存溢出  ：增加Driver的内存



2）map过程产生大量对象导致内存溢出：这种溢出的原因是在单个map中产生了大量的对象导致的，具体做法可以在会产生大量对象的map操作之前调用repartition方法 



3）数据不平衡导致内存溢出  ：调用repatition重分区



4）shuffle后内存溢出  ：在Spark中，join，reduceByKey这一类型的过程，都会有shuffle的过程 ，shuffle后，单个文件过大导致的 ，设置分区



5）使用rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)代替rdd.cache() 

- rdd.cache()和rdd.persist(Storage.MEMORY_ONLY)是等价的，在内存不足的时候rdd.cache()的数据会丢失，再次使用的时候会重算，而rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)在内存不足的时候会存储在磁盘，避免重算，只是消耗点IO时间。



## 10 spark中的数据倾斜的现象、原因、后果

1）数据倾斜的现象

多数的task执行很快，少数的task执行很慢；或者等待很长时间后提示你内存不足，执行失败



2）数据倾斜的原因

a：数据问题：key的设置不合理或者key本身分布不均匀（例如包含大量的空值）

b：spark的问题：shuffle的并行度不够，计算方式有问题



3）数据倾斜的后果

spark的stage执行时间受限与最后那个执行完的task，因此运行缓慢的任务会拖慢整个程序的运行速度

过多的数据在同一个task上执行，造成executor压力过大，同时其他节点空闲，资源利用率低下



## 11 如何解决spark中的数据倾斜问题

**1）数据问题造成的数据倾斜：**可以通过抽样找出处理异常的key值：

1.1）：如果是无效的数据则直接过滤掉，或者随机打散到其他值上

1.2）：如果是有效数据

a：隔离执行，将异常的key过滤出来单独处理，之后和正常数据进行合并(union)操作

b：将key先添加随机值，进行操作后，去掉随机值，然后再进行一次操作

c：使用reduceByKey来代替groupByKey，因为reduceByKey可以执行本地化的合并

d：使用map join，对Map 端部分聚合

>**如果使用reduceByKey因为数据倾斜造成运行失败的问题。具体操作流程如下:**  
>
>(1) 将原始的 key 转化为 key + 随机值(例如Random.nextInt)
>
>(2) 对数据进行 reduceByKey(func)
>
>(3) 将 key + 随机值 转成 key
>
>(4) 再对数据进行 reduceByKey(func)
>
>**案例操作流程分析：**  
>
>假设说有倾斜的Key，我们给所有的Key加上一个随机数，然后进行reduceByKey操作；此时同一个Key会有不同的随机数前缀，在进行reduceByKey操作的时候原来的一个非常大的倾斜的Key就分而治之变成若干个更小的Key，不过此时结果和原来不一样，怎么破？进行map操作，目的是把随机数前缀去掉，然后再次进行reduceByKey操作。（当然，如果你很无聊，可以再次做随机数前缀），这样我们就可以把原本倾斜的Key通过分而治之方案分散开来，最后又进行了全局聚合



**2）spark使用不当造成的数据倾斜**

a：提高shuffle并行度：设置spark.sql.shuffle.partitions参数控制shuffle的并发度，默认为200 

b：使用map join替代reduce join：必须是小表数据量不能太大，因为要把小表数据发送给每个Executor上。

c：开启spark的反压机制：动态调整数据的接受速度和处理速度

d：JVM调优和垃圾收集策略，合并小文件，减少序列化和反序列化的开销等





## ------------Streaming--------------

## 1 Spark Streaming的数据可靠性 

有了checkpoint机制、write ahead log机制、Receiver缓存机制、可靠的Receiver（即数据接收并备份成功后会发送ack），可以保证无论是worker失效还是driver失效，都是数据0丢失。

原因是：

如果没有Receiver服务的worker失效了，RDD数据可以依赖血统来重新计算；

如果Receiver所在worker失败了，由于Reciever是可靠的，并有write ahead log机制，则收到的数据可以保证不丢；

如果driver失败了，可以从checkpoint中恢复数据重新构建。



## 2 flume整合sparkStreaming问题

**1）如何实现sparkStreaming读取flume中的数据**  

一种是拉模式：Flume将数据Push推给Spark Streaming 

一种是推模式 ：Spark Streaming从flume 中Poll拉取数据 

sparkStreaming通过拉模式整合的时候，使用了**FlumeUtils**这样一个类，该类是需要依赖一个额外的jar包（spark-streaming-flume_2.10）



**2）在实际开发的时候是如何保证数据不丢失的** 

在Flume这边配置channel，将数据落地到磁盘上，保证数据源安全性

>memory ：设置为内存的，数据接收处理的速度快，但是数据在内存中，安全机制保证不了
>
>故选择channel为磁盘存储，整个流程运行有一点的延迟性 

要想保证数据不丢失，数据的准确性，可以在构建StreamingConext的时候 ，**设置检查点和更新检查点的周期，并且开启预写日志**

```scala
//StreamingContext.getOrCreate（checkpoint, creatingFunc: () => StreamingContext）
使用StreamingContext.getOrCreate来创建StreamingContext对象,
传入的第一个参数是checkpoint的存放目录，
第二参数是生成StreamingContext对象的用户自定义函数。
```



流失计算中使用checkpoint的作用： 

- 保存元数据，包括流式应用的配置、流式没崩溃之前定义的各种操作、未完成所有操作的batch。元数据被存储到容忍失败的存储系统上，如HDFS。这种ckeckpoint主要针对driver失败后的修复。
- 保存流式数据，也是存储到容忍失败的存储系统上，如HDFS。这种ckeckpoint主要针对window operation、有状态的操作。无论是driver失败了，还是worker失败了，这种checkpoint都够快速恢复，而不需要将很长的历史数据都重新计算一遍（以便得到当前的状态）。

- 对于一个需要做checkpoint的DStream结构，可以通过调用DStream.checkpoint(checkpointInterval)来设置ckeckpoint的周期，经验上一般将这个checkpoint周期设置成**batch周期的5至10倍。**

使用write ahead logs功能 ：

- 这是一个可选功能，建议加上。这个功能将使得输入数据写入之前配置的checkpoint目录。这样有状态的数据可以从上一个checkpoint开始计算。开启的方法是把spark.streaming.receiver.writeAheadLogs.enable这个property设置为true。另外，由于输入RDD的默认StorageLevel是MEMORY_AND_DISK_2，即数据会在两台worker上做replication。**实际上，Spark Streaming模式下，任何从网络输入数据的Receiver（如kafka、flume、socket）都会在两台机器上做数据备份。如果开启了write ahead logs的功能，建议把StorageLevel改成MEMORY_AND_DISK_SER。修改的方法是，在创建RDD时由参数传入。**



使用以上的checkpoint机制，确实可以保证数据0丢失。但是一个前提条件是，数据发送端必须要有缓存功能，这样才能保证在spark应用重启期间，数据发送端不会因为spark streaming服务不可用而把数据丢弃。而flume具备这种特性，同样kafka也具备。



## 3 sparkstreaming和kafka集成的两种方式

参考：[sparkstreaming和kafka集成的两种方式](https://blog.csdn.net/weixin_39478115/article/details/78884876)

**Receive-based：**push数据，高级API，内存泄漏，高可用WAL后，数据被复制两份、

**Direct方法：**低级API，pull数据，

1）**简化并行读取：**Kafka partition对应RDD partition 

2）**高性能：**对比Receive，不需要开启WAL机制来复制两份数据，基于direct使用kafka的副本保证零数据的丢失 

3）**一次且仅一次的事务机制：**Streaming自己追踪消费offset，保存再checkpoint中，保证数据消费一次。由于数据消费偏移量是保存在checkpoint中，因此，如果后续想使用kafka高级API消费数据，需要手动的更新zookeeper中的偏移量 。也可以将zk中的偏移量保存在mysql或者redis数据库中，下次重启的时候，直接读取mysql或者redis中的偏移量，获取到上次消费的偏移量，接着读取数据。 



-----

**Receive-based：基于接收者**
算子：KafkaUtils.createStream
方法：PUSH，从topic中去推送数据，将数据推送过来
API：调用的Kafka高级API
效果：

> SparkStreaming中的Receivers，恰好Kafka有发布/订阅 ，然而：此种方式企业不常用，说明有BUG，不符合企业需求。因为：接收到的数据存储在Executor的内存，会出现数据漏处理或者多处理状况

解释：

> 使用Kafka的高层次Consumer API来实现。receiver从Kafka中获取的数据都存储在Spark Executor的内存中，然后Spark Streaming启动的job会去处理那些数据。然而，在默认的配置下，这种方式可能会因为底层的失败而丢失数据。如果要启用高可靠机制，让数据零丢失，就必须启用Spark Streaming的预写日志机制（Write Ahead Log，WAL）。该机制会同步地将接收到的Kafka数据写入分布式文件系统（比如HDFS）上的预写日志中。所以，即使底层节点出现了失败，也可以使用预写日志中的数据进行恢复。

缺点：

> 1、Kafka中topic的partition与Spark中RDD的partition是没有关系的，因此，在KafkaUtils.createStream()中，提高partition的数量，只会增加Receiver的数量，也就是读取Kafka中topic partition的线程数量，不会增加Spark处理数据的并行度。
>
> 2、可以创建多个Kafka输入DStream，使用不同的consumer group和topic，来通过多个receiver并行接收数据。
>
> 3、如果基于容错的文件系统，比如HDFS，启用了预写日志机制，接收到的数据都会被复制一份到预写日志中。因此，在KafkaUtils.createStream()中，设置的持久化级别是StorageLevel.MEMORY_AND_DISK_SER。

**Direct Approach：直接方法**
算子：KafkaUtils.createDirectStream
方式：PULL，到topic中去拉取数据。
API：kafka低级API
效果：

> 每次到Topic的每个分区依据偏移量进行获取数据，拉取数据以后进行处理，可以实现高可用

解释：

> 在Spark 1.3中引入了这种新的无接收器“直接”方法，以确保更强大的端到端保证。这种方法不是使用接收器来接收数据，而是定期查询Kafka在每个topic+分partition中的最新偏移量，并相应地定义要在每个批次中处理的偏移量范围。当处理数据的作业启动时，Kafka简单的客户API用于读取Kafka中定义的偏移范围（类似于从文件系统读取文件）。请注意，此功能在Spark 1.3中为Scala和Java API引入，在Spark 1.4中针对Python API引入。

优势：

> 1、简化并行读取：如果要读取多个partition，不需要创建多个输入DStream，然后对它们进行union操作。Spark会创建跟Kafka partition一样多的RDD partition，并且会并行从Kafka中读取数据。所以在Kafka partition和RDD partition之间，有一个一对一的映射关系。
>
> 2、高性能：如果要保证零数据丢失，在基于receiver的方式中，需要开启WAL机制。这种方式其实效率低下，因为数据实际上被复制了两份，Kafka自己本身就有高可靠的机制会对数据复制一份，而这里又会复制一份到WAL中。而基于direct的方式，不依赖Receiver，不需要开启WAL机制，只要Kafka中作了数据的复制，那么就可以通过Kafka的副本进行恢复。
>
> 3、一次且仅一次的事务机制：基于receiver的方式，是使用Kafka的高阶API来在ZooKeeper中保存消费过的offset的。这是消费Kafka数据的传统方式。这种方式配合着WAL机制可以保证数据零丢失的高可靠性，但是却无法保证数据被处理一次且仅一次，可能会处理两次。因为Spark和ZooKeeper之间可能是不同步的。基于direct的方式，使用kafka的简单api，Spark Streaming自己就负责追踪消费的offset，并保存在checkpoint中。Spark自己一定是同步的，因此可以保证数据是消费一次且仅消费一次。由于数据消费偏移量是保存在checkpoint中，因此，如果后续想使用kafka高级API消费数据，需要手动的更新zookeeper中的偏移量。也可以将zk中的偏移量保存在mysql或者redis数据库中，下次重启的时候，直接读取mysql或者redis中的偏移量，获取到上次消费的偏移量，接着读取数据。 





## 其它



**Spark Streaming:**

抽象了离散的Dstream数据流，连续的RDD，构建Dstream链，提交Job作业运行。

批处理间隔、滑动窗口的长度和大小，批处理间隔<滑动窗口的大小

**反压机制：**动态调整数据的接受速度和处理速度、优化点

**输入源：**text、socket、flume、kafka、network、file

**Spark与Spark streaming的关系？：**转化、映射：构建的Dstream链会被构建成RDD链

**Persist()与cache()：**缓存级别

**Shuffle机制：**拉取上一个stage的数据

Job=ResultStage+ n个ShuffleMapStage，（ResultTask，ShuffleMapTask）

**HashShuffle：**缺点是数据量大的时候，内存中产生大量的Bucket缓冲区，占内存，GC

（1.6前）一个ShuffleMapTask对应ReduceTask-->bucket桶（file），可复用

**SortShuffle：**

1） **ByPassMergeSortShuffleWrite**: 适合在Reducer数量不大，又不需要在map端聚合和排序，则将数据是直接写入文件，缺点：数据量较大的时候，网络I/O和内存负担较重

2） **SortShuffleWrite：**适用于大数据量，引入了外部排序，可支持Map端的聚合，内存不够可溢写

3） UnsafeShuffleWriter：涉及到统一内存管理

**Spark和MR的区别？**

使用场景，速度，容错，Shufle结果？

**Spark的shuffle和MR的Shuffle的异同？：**磁盘、排序、内存

**相同：**都是获取map数据，发送给reduce，原理一样。

**区别：**

1）逻辑：MR的shuffle为了方便GroupByKey，回排序；Spark默认不排序

2）数据流：MR只能从一个Map Stage 中得到数据，Spark可以从多个Map Stage中得到数据，基于DAG图，可复杂操作

3）性能：MR的Shuffle单一；Spark更全面，不同的操作和write方法

4）获取数据的粒度：MR是粗粒度，buffle满了才combine；Spar可细粒度，即使fetch

5）shuffle排序：MR是基于Key的sort-based；Spark有sort-based和hash-based

**Spark的基本工作流程？**

作业调度：DAGSheduler，是高层调度模块，作业的拆分然后以taskset形式提交给任务调度模块；有两个接口：taskSheduler(交互，具体任务的调度和运行)、shedulerBackend(与底层资源交互，具体任务的资源分配)

任务调度：TaskSheduler，将TaskSET提交给worker运行，每个Executor运行什么Task就是在此处分配的.

Executor：执行任务，创建taskTunner类，提交给线程池运行。

**Spark的分区器？**

HashPartitioner：key哈希处于分区个数==》分区ID 缺点：数据倾斜

RangePartitioner：每个分区的数据均匀，分区和分区有序，但分区内无序（先抽样数据-排序-取最大key值-得到数组rangBound变量；按照数组进行key值的划分）

**Spark Join的三种方式？**

广播Join（Broadcase join）:将维度表发给每个节点，executor存储维度表的全部数据；缺点：只能广播小表

Shuffle hash join：按照join key分区，对分区数据进行join，小表分区构造hash表，再根据大表中记录进行匹配 

Sort Merge join：两个大表join，先分别按照join key进行shuffle，保证join keys值相同的记录分在相同的分区，然后分区排序，

**Streaming核心概念：**DStream(连续)、Input DStream(数据源) 、Receivers(接受对象)、Transformation(转换操作)、output operations(输出操作)、UpdataStateByeKey(带状态的算子)、window(窗口的使用=窗口的长度、滑动窗口的间隔)

**Streaming整合Kafka：**

**Receive-based：**push数据，高级API，内存泄漏，高可用WAL后，数据被复制两份、

**Direct方法：**低级API，pull数据，

1） 简化并行读取：Kafka partition对应RDD partition  2）高性能：对比Receive，不需要开启WAL机制来复制两份数据，基于direct使用kafka的副本保证零数据的丢失 3）一次且仅一次的事务机制：Streaming自己追踪消费offset，保存再checkpoint中，保证数据消费一次。由于数据消费偏移量是保存在checkpoint中，因此，如果后续想使用kafka高级API消费数据，需要手动的更新zookeeper中的偏移量

**两种创建StreamingContext的方式？** 1）SparkConf() 2）SparkContext()

**StreamingContext定义后需要？**通过DStream创建输入数据源，算子操作，计算逻辑，开启start()，等待终止awaitTermination()。

**重分区Spark算子：**coalesce：分区但不进行Shuffle，repartition：进行Shuffle，调用coalesce，参数为true

**Spark SQL执行流程：**逻辑计划—物理计划

Sql—Sql解析器---没有Shema信息的逻辑计划—Catalyst分析器—有Schema信息的逻辑计划—优化器优化—优化的逻辑计划—添加策略—转化成物理计划—执行—DataFrame

**Spark SQL特点：**数据兼容、组件扩展、性能优化：内存列式存储，动态字节码生成，内存缓存数据、支持多语言

**调度管理：**Job、Stage、TaskSet、Task

DAGScheduler—划分stage—具有依赖关系的TaskSet，逻辑（作业）调度模板

TaskScheduler—提交TastSet—TaskSetManager

TaskSetManager：具体任务集的内部调度任务

**调度池：**FIFO（根据StageID顺序调度）、Fair（设置最小任务数，权重：根pool-子pool-TaskSetManager）

**Job调度流程：**

1） 划分Stage：从后向前遍历RDD依赖链，以Shuffle为依据划分Sage

2） 生成Job，提交Stage：判断父依赖是否而可用，不可用则迭代尝试，不成功则加入等待队列中

3） 任务集的提交：stageàTaskSet，DAGScheduler通过TaskScheduler接口提交TS，触发生成TashSetManager实例来管理这个TaskSet的生命周期

4） 任务作业完成状态的监控：DGASchedular通过TaskScheduler的接口暴露的一系列回调函数来获取其状态

5） 任务结果的获取：任务类型不同，FinalStage返回DAGSchedular运算结果本身，ShuffleMapTask返回给DAGSchedular的是一个MapStatus对象；结果小则直接放到DirectTaskResult对象中，大于特定尺寸，则序列化为bolck块，存放在BlockManager内。

**调度模块：**

DAGScheduler：接受用户提交的Job，将Job根据类型划分为不同的Stage，并在每一个Stage内产生一系列的Task，向TaskScheduler提交Task。

TaskSetManager：完成任务内部的调度逻辑

TaskScheduler：接受DAGScheduler的任务集，然后向集群分发任务，并且调用后台任务

Stage：根据RDD依赖关系，划分Stage，形成DAG图；Shuffle操作作为Stage的边界(ShuffleMapStage、ResultStage)

**存储管理：**

1） Shuffle数据持久化：Shuffle涉及了磁盘的读写和网络的传输，Shuffle性能的高低直接影响到程序的运行效率

2） 存储层storage：以Block块进行操作，RDD运算以Partition进行操作，Partition最终被映射为Block来存取和处理（分区和数据块一一对应，每个RDD都有一个唯一的ID，每个分区都有唯一的索引，以**ID+索引号**作为数据块名便自然的建立起了分区与块的映射。）

**Spark性能优化：**

程序优化：减少shuffle操作。设置缓存cache和缓存级别，优化Partition，重新分区、合并小文件，降低并发量

资源配置：设置executor资源参数，查看日志，内存优化/GC优化。

其他优化：压缩处理、序列化、设置共享变量等

**Spark SQL性能调优：**通过缓存数据(缓存表，构建一个内存中的列格式缓存表，Spark SQL仅扫描需要的列，并自动调整压缩比，使内存使用率和GC压力最小)、调优参数、增加并行度提升性能(并行加载数据)

**Spark Streaming性能调优：**

提升数据接收并行度、提升数据处理的并行度，减少序列化和反序列化负担，减少任务提交和分发开销(粗粒度运行)，优化内存使用(设置缓存级别、及时清理持久化的RDD、采用并发垃圾收集策略)，设置合适的批次大小

**Spark组件：**

1）负责作业提交的Client端、2）控制集群的主控节点Master、3）负责作业控制的Driver进行组成、4）负责集群资源管理的ClusterManager、5）执行具体任务的Worker节点、6）执行单元Executor、

**备注：**Executor中有Task任务池和BlockManager（存储管理：备份、支持重试和故障恢复）

**Spark on Yarn的执行机制：**

1） client—作业信息—提交给ResourceManager 2）RM在NodeManager汇报时，让NM启动SparkAppMaster

3）SparkAppMaster启动后，初始化作业，向RM申请资源 4）RM通过RPC在NM启动Executor，执行任务，并向AppMaster汇报任务状态。

 











