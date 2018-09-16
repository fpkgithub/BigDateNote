



## 1 HBase特点有那些？

参考：[Hbase问题汇总1](https://www.jianshu.com/p/57be566c1873)

存储规模大：单表可达数十亿行，数百万列 

面向列：存储和检索均面向列 

无模式：同一表中的不同行可以由截然不同的列 

存储稀疏：空列并不占用存储空间，表可以设计的非常稀疏 

多版本：每个单元（cell）中的数据可以由多个版本，默认情况下版本号是单元格插入的时间戳 

数据类型单一：数据都是字符串，没有类型（所有数据都被转换为字节数组） 



## 2 HBase VS HDFS VS Hive

1）HBase VS HDFS
两者都有很好的容错性和扩展性
HDFS适合离线批处理场景、不支持随机查找、不支持数据更新
HBase是对HDFS很好的补充

2） HBase VS Hive
两者都依赖于HDFS作为数据存储支持，都有表、数据库的概念
Hive面向计算层面的，为统计分析而生的
HBase面向存储，要满足实时查询需求



## 3 HBase的缺点

参考：[Hbase问题汇总1](https://www.jianshu.com/p/57be566c1873)

1 不能支持条件查询，仅能通过行键和行键序列来检索数据 

>要进行条件查询只有两种方式： 
>
>（1）.设计合适的行键（通过行键直接定位到数据所在的位置）； 
>
>（2）.通过Scan方式进行查询，Scan可设置起始行和结束行，把这个搜索限定 在一个区域中进行； 

2 仅支持行级（单行）事务（HBase的事务是行级事务，可以保证行级数据的原子性、一致性、隔离性以及持久性） 

>HBase分别提供了行锁和读写锁来实现行级数据、Store级别以及Region级别的并发控制。除此之外，HBase还提供了MVCC机制实现数据的读写并发控制。MVCC，即**多版本并发控制技术**，它使得事务引擎不再单纯地使用行锁实现数据读写的并发控制，取而代之的是，把行锁与行的多个版本结合起来，经过简单的算法就可以实现非锁定读，进而大大的提高系统的并发性能。HBase正是使用**行锁 ＋ MVCC**保证高效的**并发读写**以及**读写数据**一致性。 详情：[HBase 事务和并发控制机制原理](https://blog.csdn.net/u012164361/article/details/72758012) 



## 4 HBase的体系结构？

**Client：**

- 包含访问HBase的接口，并维护cache来加快对HBase的访问，比如region的位置信息
- Client读写HBase上数据不需要与Master交互，只需要寻址访问Zookeeper和RegionServer

**Master：**

- 为RegionServer分配Region
- 发现失效的RegionServer，并重新分配其上的Region
- 在Region 分裂后，负责新Region的分配
- 负责RegionServer的负载均衡
- 负责管理用户对表的增删改查操作
- 仅仅维护Table和Region的元数据信息，负载很低，

**Regionserver：**

- 负责用户的IO请求
- 负责region的分裂

**Zookeeper：**

- 保证任何时候，集群中只有一个Active Master
- 存储所有Region的寻址入口
- 实时监控RegionServer的状态，将RegionServer的上下线信息实时通知给Master



## 5 Hbase逻辑结构

**表（Table）：**
hbase在表中组织数据。表名是字符串和字符的组合，可以在文件系统路径中使用。

**行（Row）：**
在表中数据依赖于行来存储，行通过行键来区分。行键没有数据类型，通常是一个**字节数组。**每一行rowkey必须是唯一的。可以使用（哈希值+时间戳来作为行键）

**列族（Column Family）：** 行中的数据通过列族来组织。列族也暗示了数据的物理排列。**所以列族必须预先定义，**并且不容易被修改。每行都拥有相同的列族，可能有些行的数据为空。列族是字符串和字符的组合，可以在文件系统路径 中使用。 

**单元（Cell）：**
通过行键和列唯一确定的一个存储单元；
Cell中可能包含多个版本的数据；
**Cell中数据是没有类型的，全部是字节码形式存储。**

**版本（Version）：**
每行数据可以有多个版本
Version默认是TimeStamp（当前系统时间），在数据写入时自动赋值；
Version也可以由客户显式赋值；
不同Version的数据按Version**倒序排序**，即最新的数据排在最前面；
获取数据时不指定Version，默认取最新的数据；

![img](https://images2015.cnblogs.com/blog/1123009/201703/1123009-20170313151622870-217153522.png) 



## 6 主要操作？？？	

get ：**返回结果有序，按指定rowkey获取唯一一条记录**

scan ：**返回结果有序，按指定条件获取一批记录** 

put（添加或者更新，更新不会删除旧数据，会以版本号标识，先缓冲区--->RPC发送） 

delete：建立一个墓碑标志，不直接删除，在合并Compact时再进行删除。 

其缺陷：删除版本T之后，在合并前，有添加了新数据版本号低于T，则get时无法查询到。解决办法：使用时间戳作为版本号 

注意：

delete：删除记录后，会写日志，可以回滚

truncate：执行速度块，删除后就不可以回滚了

drop：删除表结构及所有数据，并将表所占用的空间全部释放。 

>如果想删除表，当然用drop； 
>
>如果想保留表而将所有数据删除，如果和事务无关，用truncate即可；
>
>如果和事务有关，或者想触发trigger，还是用delete；
>
>如果是整理表内部的碎片，可以用truncate跟上reuse stroage，再重新导入/插入数据。

```sql
[HBase常用命令:https://www.cnblogs.com/shadowalker/p/7350484.html]
//创建表 表名：User  列簇：info
create 'User', 'info'

//查看所有的表  list
//查看表的详情  describe

//插入数据
put 'User','row1','info:name','xiaoming'
put 'User','row2','info:age','18'

//查询数据
hbase(main):008:0> get 'User', 'row2'
COLUMN                                  CELL
info:age                               timestamp=1502368069926, value=18
1 row(s) in 0.0280 seconds

hbase(main):028:0> get 'User', 'row3', 'info:sex'
COLUMN                                  CELL
info:sex                               timestamp=1502368093636, value=man

hbase(main):036:0> get 'User', 'row1', {COLUMN => 'info:name'}
COLUMN                                  CELL
 info:name                              timestamp=1502368030841, value=xiaoming
1 row(s) in 0.0120 seconds

//扫描所有记录
scan 'User'

//扫描前2条
scan 'User', {LIMIT => 2}

//指定扫描  
scan 'User', {STARTEND => 'row2', ENDROW => 'row3'}
还可以添加timeRange和filter等高级功能；STARTROW,ENDROW必须大写，否则报错;查询结果不包含等于ENDROW的结果集

//统计表记录数
count 'User'

//删除列
delete 'User','row1','info:age'
//删除行
delete 'User','row2'
//删除表中所有数据
truncate 'User'
//禁用表，启用表，删除表
```



## 7 HBase的物理结构

HBase中所有数据文件存储在HDFS，主要有两种文件类型：

- HFile：Hadoop**的二进制格式文件，存储KeyValue数据**，可以建立索引，提高读的效率， HFile默认是十亿字节进行拆分，Trailer（指针，指向其他数据块）、Data Block Index、Data Block（魔数：信息校验+Key/Value 可以被压缩） 
- Hlog：HBase中WAL（Write Ahead Log） 的存储格式，物理上是Hadoop的Sequence File，预写日志，将RegionServer插入和删除过程记录操作内容的日志，可用于灾难恢复；顺序追加写，减少磁盘寻址，定期删除。 

StoreFile是对HFile的轻量级包装，即StoreFile底层就是HFile；

一个RegionServer只有一个Hlog



## 8 HBase表物理存储

[HBase中Region, store, storefile和列簇的关系](https://www.cnblogs.com/mrxiaohe/p/5271578.html) 

![img](https://upload-images.jianshu.io/upload_images/5959612-363ba61452e2d978.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/885/format/webp) 



Table中所有行都按照rowkey的字典序排序，Table 在行方向上分为多个Region 

Region由一个或者多个Store组成，**每个Store保存一个column family（对应着一个目录），** 每个Store又由一个MemStore和0至多个StoreFile组成 

- MemStore：用户写入的数据首先放入MemStore，MemStore满了会Flush成一个StoreFile（底层实现是HFile）
- StoreFile：StoreFile文件数量增长到一定阈值，会触发Compact操作，将多个StoreFile合并成一个StoreFile

Compact：合并过程会进行版本合并和数据删除，HBase只增加数据，所有更新和删除都是在后续的compact过程中进行的

异步写入：用户写操作只要进入内存就可以立即返回，保证了HBase I/O的高性能 

**一个RegionServer上有一个BlockCache和N个Memstore** 



## 9 HLog的日志恢复Hlog Replay

- RegionServer挂掉，MemStore中数据会丢失。Master通过Zookeeper感知到RegionServer故障
- Master将不同Region的Log数据进行拆分，并将失效的Region重新分配；
- 领取到待恢复Region的RegionServer在Load Region时，会Replay HLog中的数据到MemStore；



## 10 Hbase的读数据的过程

参考：[Hbase问题汇总3](https://www.jianshu.com/p/c92b1bf6ee4b)

>hmaster启动时候会将hbase 系统表-ROOT- 加载到 zookeeper cluster，通过zookeeper cluster可以获取当前系统表.META.的存储所对应的regionserver信息。 
>
>regionserver 向zookeeper注册，提供hbase regionserver状态信息（是否在线） 

1）客户端调用get或者scan进行查询

2）定位region位置：去zookeeper上寻找表的入口地址---在root中找到meta表region地址---在meta表中检索region信息

3）根据region的入口地址，如region中查找。构造扫描器，先从memstore中查找---backcache中查找---strorfile中查找



## 11 Hbase存储数据的过程

1）**客户端提交写请求，先将数据写入缓存，判断缓存是否满，若满则提交数据**；

2）客户端重构数据，建立映射：定位put属于哪一个region，其次判断region属于哪个regionserver，最后**构建put与regionserver之间的映射**，接下来批量提交给regionserver。 

3）客户端提交请求：客户端向各个regionserver并发提交写请求，使用线程池方式并等待结果，若有失败的结果加入队列重试。 

4）服务器端根据region名字获得region对象，**获取锁并更新时间戳，写入memstore，同时更新WAL日志。**



**Memstore主要用来写，BlockCache主要用于读** ；**一个RegionServer上有一个BlockCache和N个Memstore** 



## 12 Hbase删除数据的过程

**设置标志位--->移植文件夹缓冲区--->大合并时彻底删除** 

当接收到数据删除指令后，系统并没有立即删除HFile中存储的数据，而是设置一个标志位标志其被删除（在HDFS中数据删除时被移到/trash文件夹缓冲区），此时系统会根据标志位响应客户端的访问请求，待系统的下一次大合并(major campaction)将被标志的数据块删除，这才算彻底的完成数据的删除。 



## 13 Hbase合并介绍

主要起到如下几个作用：
1）合并文件
2）清除删除、过期、多余版本的数据
3）提高读写数据的效率

Minor & Major Compaction的区别：（小合并和大合并）
1）Minor操作只用来做部分文件的合并操作以及包括minVersion=0并且设置ttl的过期版本清理，不做任何删除数据、多版本数据的清理工作。
2）Major操作是对Region下的HStore下的所有StoreFile执行合并操作，最终的结果是整理合并出一个文件。



**合并的出发时机：**

- Memstore Flush
- 后台线程周期性检查
- 手动触发。



## 14 Hbase的Log-Structured Merge Tree架构模式？

**日志合并结构树：**将数据的修改增量保持在内存中，达到指定的大小限制后将这些修改操作批量写入磁盘，读取是需要合并磁盘中的历史数据和内存中的最近修改操作；主要是为了那些长期具有高频率更新的文件提供低成本的索引机制。LSM树的优势：有效地规避磁盘随机写入问题，但读取时需要访问较多的磁盘文件。

用户数据写入先写WAL，再写缓存，满足一定条件后缓存数据会执行flush操作真正落盘，形成一个数据文件HFile。随着数据写入不断增多，flush次数也会不断增多，进而HFile数据文件就会越来越多。然而，太多数据文件会导致数据查询IO次数增多，因此HBase尝试着不断对这些文件进行合并，这个合并过程称为Compaction。

Compaction会从一个region的一个store中选择一些hfile文件进行合并。
先从这些待合并的数据文件中读出KeyValues，再按照由小到大排列后写入一个新的文件中。之后，这个新生成的文件就会取代之前待合并的所有文件对外提供服务。



## 15 块缓存BockCache

Memstore主要用来写，BlockCache主要用于读 。

读请求先到Memstroe中查数据，查不到就到BlockCache中查，再查不出就会到磁盘上读，并把读的结果放到BlockCache中； 

BlockCache采用的是LRU策略，当块缓存达到上限后，会启动淘汰机制，淘汰掉最老的一批数据。 

**一个RegionServer上有一个BlockCache和N个Memstore** 

 

## 16 HBase优化

参考：[HBase优化](https://www.jianshu.com/p/1a8820c0f5ca)

**1、垃圾回收优化 **
使用CMS垃圾回收机制

**2、数据压缩**
GZIP、Snappy、LZO，推荐Snappy

**3、MemStore缓存配置和BlockCache配置**

**4、Region拆分和合并**
预建分区，避免自动split，提高hbase响应速度；同时调整分区的大小

**5、Region均衡**
避免出现Region热点现象，按照table级别进行balance

**6、尽量只用1-3个列族**
定期建表，如每月中旬建立下一个月的表，表名中含有年月

**7、如果数据量特别大，不能只有1个表 、确定按年/季/月建表** 

8、布隆过滤器设置（行级、列簇级），批量的读(`get(List<GET>)`)写

9、关闭restlutScanner：restlutScanner会存储服务器扫描的结果，不关闭会在一定时间内保持连接，浪费资源

10、通过HTablePool来访问：保证现场安全，因为HTable不是线程安全的额，HTablePool可以解决存在的线程安全问题，可以复用



## 17 行键RowKey设计

参考：[HBase优化](https://www.jianshu.com/p/1a8820c0f5ca)

Hbase没有数据类型，所有的数据都被转化为了字节数组

**1、保证唯一性** 

**2、长度设计不应过长**（加快再内存缓存时进行检索的效率，减少存储空间） ：a：数据会先缓存在MemStore中，太长会占用过多内存，降低资源利用率、b：rowkey是冗余存储，最终都被存储在HFile文件中，HFile的文件是按照keyvalue存储的；

**3、散列：**（散列+时间戳 或者 MD5加密后取前六位） ：避免热点访问造成性能下降

数据按照rowkey的字典序顺序存储； 设计rowkey时，充分利用排序存储特性，将经常一起读取的行存储到一起； 不要把业务发生时间直接作为rowkey，导致全部存储到一个regionserver中，可以在时间戳前面加上散列值，设计成“散列值+时间戳”的形式 



## 18 列簇设计原则

参考：[HBase优化](https://www.jianshu.com/p/1a8820c0f5ca)

**1：列簇不宜过多，**1~3个。跨列簇查询效率低下。同一个列簇的数据放在一起的，因为flush和compact是以region为单位的，如果一个Region中的Memstore过多，每次flush的开销会很大，同时一个flush操作也会影响其他不相干的列簇flush。

**2：列簇的名字不宜过长** 

**3：将经常一起查询的列放在一个列簇中** 

备注：因为列簇没有元数据的描述，所以想获取列簇的完整列名，则需要处理所有的行 





