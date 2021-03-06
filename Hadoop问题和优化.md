[TOC]



## ----------------MR----------------

## 1 什么是MapReduce

是一种分布式处理数据的计算模型，采用分而治之的思想，把处理过程高度抽象为两个函数：map()函数和reduce函数，map函数将任务分解成多个任务，reduce负责把分解后的多个任务结果汇总起来。 



## 2 MapReduce的工作过程

[mapreduce框架详解](https://www.jianshu.com/p/6fcc5c65fe10)

MR程序的执行过程主要分为三步：Map阶段、Shuffle阶段、Reduce阶段 

![img](https://upload-images.jianshu.io/upload_images/5959612-b0bc41948b1143e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/853/format/webp) 

**1、Map阶段**

**1）分片（Split）**：map阶段的输入通常是HDFS上文件，在运行Mapper前，FileInputFormat会将输入文件分割成多个split ——1个split至少包含1个HDFS的Block（默认为64M）；然后每一个分片运行一个map进行处理。

**2）执行（Map）：**对输入分片中的每个键值对调用`map()`函数进行运算，然后输出一个结果键值对。 

- Partitioner：对`map()`的输出进行partition，即根据key或value及reduce的数量来决定当前的这对键值对最终应该交由哪个reduce处理。默认是对key哈希后再以reduce task数量取模，默认的取模方式只是为了避免数据倾斜。然后该key/value对以及partitionIdx的结果都会被写入环形缓冲区。

**3）溢写（Spill）：** map输出写在内存中的环形缓冲区（字节数组），默认当缓冲区满80%，启动溢写线程，将缓冲的数据写出到磁盘。 

- Sort：在溢写到磁盘之前，使用快排对缓冲区数据按照partitionIdx, key排序。（每个partitionIdx表示一个分区，一个分区对应一个reduce）
- Combiner：如果设置了Combiner，那么在Sort之后，还会对具有相同key的键值对进行合并，减少溢写到磁盘的数据量。

**4）合并（Merge）：**溢写可能会生成多个文件，这时需要将多个文件合并成一个文件。合并的过程中会不断地进行 sort & combine 操作，最后合并成了一个已分区且已排序的文件。

**2、Shuffle阶段**

广义上Shuffle阶段横跨Map端和Reduce端，在Map端包括Spill过程，在Reduce端包括copy和merge/sort过程。通常认为Shuffle阶段就是将map的输出作为reduce的输入的过程 

1）Copy过程：Reduce端启动一些copy线程，通过HTTP方式将map端输出文件中属于自己的部分拉取到本地。Reduce会从多个map端拉取数据，并且每个map的数据都是有序的。

2）Merge过程：Copy过来的数据会先放入内存缓冲区中，这里的缓冲区比较大；当缓冲区数据量达到一定阈值时，将数据溢写到磁盘（与map端类似，溢写过程会执行 sort & combine）。如果生成了多个溢写文件，它们会被merge成一个**有序的最终文件**。这个过程也会不停地执行 sort & combine 操作。

3、**Reduce阶段**：

Shuffle阶段最终生成了一个有序的文件作为Reduce的输入，对于该文件中的每一个键值对调用`reduce()`方法，并将结果写到HDFS。



## 3 MapReduce的优化

[MapReduce过程详解及其性能优化](https://www.jianshu.com/p/9e4d01b74600)

分块优化，减少网络传输数据，使用本地数据运行map任务。

设置Map和Reduce的数量，提前合并小文件（正常的map数量的并行规模大致是每一个Node是10~100个。 ）

设置环形缓冲区大小

自定义partition机制，防止数据倾斜，设置环形缓冲区大小

combine时一个本地化的reduce操作，对相同的key做一个合并操作，提高带宽的利用率

数据压缩传输

Shuffle阶段，Reduce任务通过HTTP向各个Map任务下载获取的数据 ，默认情况下，每个Reducer只会有5个map端并行的下载线程在从map下载数据 在Reducer内存和网络都比较好的情况下，可以调大该参数 ，并且加大超时时间，防止网络不好，造成误判为失败的情况

设置Reduce数量：正确的reduce任务的个数应该是0.95或者1.75 （节点数*参数值）

在Reduce段，拉取来的数据，也会进行Merge Sort，此时有三种方式：

1）内存到内存（memToMemMerger） 默认关闭

 2）内存中Merge（inMemoryMerger） 达到阈值，则内存Merge然后溢写磁盘

3）磁盘上的Merge（onDiskMerger） 磁盘上进行合并



## 4 combine、partition和shuffer的区别？

**combine** ：把同一个key的键值对合并在一起 ，本地化的Reduce，（**类型一致 不影响 累加 最大值** 才能使用）

**partition** ：数据规约，根据key或value及reduce的数量来决定当前的这对键值对最终应该交由哪个reduce处理 ，默认是对key哈希后再以reduce task数量取模，默认的取模方式只是为了避免数据倾斜。 默认使用HashPartition，可以自定义partition。

**shuffle：**是map和reduce之间的过程，包含了两端的combine和partition ，将map端的输入作为reduce的输出。



## 5 shuffle 是什么？ 怎么调优？

shuffle将map的输出作为reduce端的输入，包括map端的combine和partition，以及reduce端的copy和combine； 其目的就是：完整地从map task端拉取数据到reduce 端；在跨节点拉取数据时，尽可能地减少对带宽的不必要消耗，减少磁盘IO对task执行的影响。 

调优：减少I/O操作和提高网络传输效率 

## 6 TextInputFormat作用和实现？

InputFormat会在map操作之前对数据进行两方面的预处理： 

1 是getSplits，返回的是InputSplit数组，**对数据进行split分片，**每片交给map操作一次； 

2 是getRecordReader，返回的是RecordReader对象，**对每个split分片进行转换为key-value键值对格式传递给map函数** 

常用的InputFormat是TextInputFormat，使用的是LineRecordReader对每个分片进行键值对的转换，以行偏移量作为键，行内容作为值；

自定义类继承InputFormat接口，重写getRecordReader和getSplits方法



## 7 Hadoop中的数据压缩

**数据压缩优点：**减少存储磁盘空间，降低单节点的磁盘IO。 加快网络传输的效率。 

**缺点：**需要花费额外的时间/CPU做压缩和解压缩计算 

**压缩方式：** 压缩大小 压缩速度 是否可分割

![img](https://upload-images.jianshu.io/upload_images/5959612-59a8b7eb9ad05271.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/570/format/webp) 



## 8 MapReduce的缺陷

编写复杂：复杂运算许多写多个MR，一般都是基于Hive或者Pig 

不能写入到已经存在内容的目录：这实际上不算是MapReduce的缺陷，它是HDFS的一个特点．HDFS的特色就是修改数据不方便 

输出结果的形式单一：Reduce阶段的输出，就是一个<Key, Value>的键值对，然而，很多时候，我们的输出结果不是这么简单的

开发调试不方便：Hadoop将System.out.println()语句的输出，都收集起来了，作为自己的输出，要看的话，需要到JobTracker的WebUI中，查看日志． 



## 9 MapRuduce在Yarn中的工作机制

![img](https://upload-images.jianshu.io/upload_images/5959612-7831d525b3c24293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/598/format/webp)



1）**客户端**向resourcemanager**发送job请求**，客户端产生Runjar进程与resourcemanager通过rpc通信

2）resourcemanager向客户端**返回Job相关资源的提交路径以以及jobID**

3）客户端将Job相关的资源**提交到相应的共享文件系统的路径下**（eg.HDFS）

4）客户端向resourcemanager**提交job**

5）resourcemanager通过调度器在nodemanager**创建一个容器**，并在容器中**启用 MRAppMaster进程**（该进程由resourcemanager启动）。

6）该MRAppMaster进程**对作业进行初始化**，创建多个对象对作业进行跟踪。

7）MRAppMaster从共享文件系统中**获得计算到的输入分片**（只获取分片信息，不需要jar等相关资源），为每一个分片**创建一个map以及指定数量的reduce对象**。之后 MRAppMaster决定如何运行构成mapreduce作业的各个任务，如果作业很小，则与 MRAppMaster在同一个JVM上运行

8）若作业很大，则MRAppMaster会为所有map任务和reduce任务向resourcemanager**发起申请容器资源请求。**请求中包含了map任务的数据本地化信息以及输入分片等信息。

9）Resourcemanager为任务分配了容器之后，MRAppMaster就通过与NodeManager通信启动容器，由**MRAppMaster负责分配在那些nodemanager上运行map(即 YarnChild进程)和reduce任务。**

10）运行map和reduce任务的nodemanager从系统中**获取job相关资源，**包括 jar文件，配置文件等。

11）**运行map和reduce任条。**

12）关于状态的检测与更新不经过resourcemanager：**任务周期性的向MRAppMaster汇报状态及进度，**客户端每秒钟通过查询一次MRAppMaster获取状态更新信息。

---

## ---------------HFDS---------------



## 1 HDFS体系结构

HDFS 采用Master/Slave的架构来存储数据，该架构主要由四个部分组成 

**HDFS Client** 

> 文件切分：文件上传 HDFS 的时候，Client 将文件切分成 一个一个的Block，然后进行存储 
>
> 与 NameNode 交互：获取文件的位置信息 
>
> 与 DataNode 交互：读取或者写入数据 
>
> Client 提供一些命令来管理HDFS：比如启动或者关闭HDFS Client 可以通过一些命令来访问 HDFS 

**NameNode** 

>master，一个管理者，不实际存储数据 
>
>1、文件系统的命名空间：文件系统目录树+文件/目录信息+文件数据块索引 
>
>2、数据块之间的对应关系：启动时动态构建，以 **命名空间镜像文件** 和 **编辑日志文件** 形式存储在内存中
>
>存在单点故障：使用备用（standby）名字节点 
>
>为了增强横向扩展解决内存的问题：联邦HDFS机制 

**Datanode** 

>存储：Datanode以存储数据块(Block)的形式保存HDFS文件
>
>响应读写请求：同时Datanode还会响应HDFS客户端读、写数据块的请求
>
>汇报信息：Datanode会周期性地向Namenode上报（心跳信息、 数据块汇报信息(BlockReport )、 缓存数据块汇报信息(CacheReport) 增量数据块块汇报信息。 ）
>
>响应Namenode指令：Namenode会根据块汇报的内容,修改Namenode的命名空间(Namespace),同时向Datanode返回名字节点指令。Datanode会响应Namenode返回的名字节点指令,如创建、删除和复制数据块指令等。

**SecondaryNameNode** 

>辅助NameNode，分担其工作量 定期合并fsimage和edits，并推送给NameNode，紧急情况下，可辅助恢复NameNode 





## 2 Fsimage和Editlog合并过程？

![img](https://upload-images.jianshu.io/upload_images/5959612-c78b6423e1f5fe86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/762/format/webp) 

> 1 secondarynamenode周期性通过edits文件大小，当达到合并的阈值后，Namenode停止使用 edits文件，并生成一个新的临时edits.new文件
>
> 2 secondarynamenode通过http的get方式获取namenode节点上的edits和fsimage文件
>
> 3 secondarynamenode将fsimage载入内存并逐一执行edits文件中的操作
>
> 4 执行完毕后，生成fsimages.ckpt，会向namenode发送http请求，通知namenode来获取fsimage.ckpt文件
>
> 5 NameNode更新fsimage文件中的记录检查点执行时间，改名为fsimage文件
>
> 6 edits.new文件更名为edits文件



## 3 HDFS的HA高可用

HDFS的高可用(High Availability, HA)方案就是为了解决Namenode的单点故障而产生的。 在HA HDFS集群中会同时运行两个Namenode 

- 一个作为活动的(Active) Namenode,
- 一个作为备份的(Standby) Namenode.

备份Namenode的命名空间与活动Namenode是实时同步的,所以当活动Namenode发生故障而停止服务时,备份Namenode可以立即功换为活动状态,而不影响HDFS集群的服务。 

**为了保证命名空间和编辑日志的数据一致性：**第一关系链的一致性 

独立运行日志节点（**JournalNodes, JNS**）通信 ，当Active Namenode执行了修改命名空间的操作时，它会定期将执行的操作记录在editlog中，并写入JNS的多数节点中，而Standby Namenode会一直监听JNS上editlog的变化，如果发现editlog有改动， Standby Namenode就会读取editlog并与当前的命名空间合并。 

当发生了错误切换时， Standby节点会先保证已经从JNS上读取了所有的editlog并与命名空间合并，然后才会从Standby状态切换为Active状态。通过这种机制，保证了Active Namenode与Standby Namenode之间命名空间状态的一致性 

**为了保证数据块之间的对应关系：**第二关系链 的一致性

Datanode会同时向这两个Namenode发送心跳以及块汇报信息。这样Active Nanenode和Standby Namenode的元数据就完全同步了，一旦发生故障，就可以马上切换，实现热备。 这样发生错误切换时， Standby节点就不需要等待所有的数据节点进行全量块汇报，而可以直接切换为Active状态。 

Standby Namenode只会更新数据块的存储信息，并不会向Namenode发送复制或者删除数据块的指令，这些指令只能由Active Namenode发送。 



**Active Namenode和Standby Namenode之间如何共享editlog日志** 

1）Hadoop2.6之前使用的共享存储时NAS（网络附属存储）+NFS（网络文件系统） 

2）2.6提供了QJM（Quorum Journal Manager）方案来实现HA共享存储 （JournalNode和QuorumJournalManagerr ）



**HA状态切换方式** ：

管理员手动通过命令执行状态切换

自动状态切换机制触发状态切换（由ZKFailoverController控制切换流程）



## 4 HDFS的块存储

为了便于文件的管理和备份，HDFS使用块作为存储系统当中的最小单位，默认一个块的大小为64MB； 当有文件上传到HDFS上时，若文件大小大于设置的块太大，则该文件会被切分存储为多个块，多个块可以存放在不同的DataNode上，整个过程中 HDFS系统会保证一个块存储在一个datanode上 。 

设置块大小的原则是：

减少硬盘寻道时间(disk seek time)、 减少Namenode内存消耗



默认分片大小与分块大小要相同：

太大，会导致map读取的数据可能跨越不同的节点，没有了数据本地化的优势 

太小，会导致map数量过多，任务启动和切换开销太大，并行度过高 



## 5 HDFS客户端读流程

![HDFS客户端读流程](https://upload-images.jianshu.io/upload_images/5959612-64c8a4ad63d268ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **打开HDFS文件：**
  调用DistributedFileSystem的方法，底层返回一个DFSInputStream对象，用来读取数据块
- **从Namenode获取Datanode地址：**
  通过ClientProtocol.getBlockLocations()方法向名字节点获取该HDFS文件起始位置数据块的位置信息（位置信息的远近进行了排序，可以选择最优的数据节点来读取）
- **连接到Datanode读取数据块：**
  客户端通过DFSInputStream.read()方法从最优的数据节点读取数据，数据会以数据包（packet）为单位从数据节点通过流式接口传送给客户端。读取完毕，则继续获取下一个文件位置...
- **关闭数据流：**
  通过DFSIputStream.close()方法关闭输入流

备注：数据包中不仅包含了数据，还包含了校验值，当校验错误时，客户端回向Namenode汇报并读取副本数据。



## 6 HDFS客户端写流程

![ HDFS客户端写流程](https://upload-images.jianshu.io/upload_images/5959612-4ed39628ff67786c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **创建文件：**客户端调用DistributedFileSystem .create方法创建新的空文件，这个方法的底层对**ClientProtocol.create远程调用**Namenode执行对应的操作，校验并创建一个空文件，记录创建操作到editlog中。**返回真正用于写数据的DFSOutputStream对象，**
- **建立数据流管道：**客户端调用DFSOutputStream.write()方法来写数据，在这之前，DFSOutputStream会首先**调用ClientProtocol.addBlock()向Namenode申请一个新的空数据块**，返回数据块的位置信息的对象。获得了数据流管道中所有数据节点的信息后，DFSOutputStream就可以建立数据流管道进行写数据块了。
- **通过数据流管道写入数据：**
  写入DFSOutputStream中的数据会先被放到**缓存队列**中，之后被切分成一个个数据包(packet)，通过数据流管道发送到数据节点。每个数据包都有个ack确认包，ack会逆序通过数据流管道回到输出流，输出流在确认所有数据包已经写入到数据节点上，就会从相应的缓存队列删除这个数据包。客户端写满一个数据块，会调用addBlock()申请新的数据块....
- **关闭输入流并提交文件:**
  完成了所有数据块的写操作之后，调用DFSOutputStream.close()方法关闭输出流；并调用ClientProtocol.complete()方法通知Namenode提交这个文件中的所有数据块，正式完成整个文件的写入流程。

**数据流管道中的数据节点出现故障时，输出流会进行如下的故障恢复：**

- 输出流中的缓存没有确认的数据包会重新加入发送队列，这种机制确保了数据节点出现故障时不会丢失任何数据，所有的数据都是经过确认的。
- 故障数据节点会从输入流管道中删除，然后通过RPC通知Namenode来分配新的数据节点到数据流管道中，并使用新的时间戳重建数据流管道。
- 数据流管道重新建立之后，输出流会调用ClientProtocol.updatePipeline()更新Namenode中的元数据。



## 7 租约管理

HDFS是一次写多次读，并且**不支持客户端的并行写**操作，为了保证写入数据互斥。**HDFS提供了租约(Lease)机制来实现** 

**租约：**是Namenode给予租约持有者(LeaseHolder，一般是客户端)在规定时间内拥有文件权限(写文件)的合同。 

在HDFS中,客户端写文件时需要先从**租约管理器(LeaseManager)申请一个租约**，成功申请租约之后客户端就成为了租约持有者，也就拥有了对该HDFS文件的独占权限，其他客户端在该租约有效时无法打开这个HDFS文件进行操作。 

Namenode的租约管理器保存了 **HDFS文件与租约**、**租约与租约持有者的对应关系，**租约管理器还会定期检查它维护的所有租约是否过期。租约管理器会强制收回过期的租约，所以租约持有者需要定期更新租约(renew)，维护对该文件的独占锁定。当客户端完成了对文件的写操作，关闭文件时，必须在租约管理器中释放租约。 

租约管理器中有两个租约过期时间：

- 软超时时间(1分钟)
- 硬超时时间(一小时),不可配置。

在超过了硬超时时间后会触发，租约恢复。

**涉及的操作有：**

添加租约---addLease()

检查租约---FsNamesystem.checkLease() 

租约更新---renewLease() 

删除租约---removeLease() 

租约检查---Monitor线程 

租约恢复---Monitor线程发起 



## 9 **租约恢复和检查过程**

**租约恢复：**

总：选择恢复主节点，修改恢复文件的租约信息，收集参与租约恢复数据块的信息，找到最优的一个数据块取其长度，申请新的数据块版本号，然后同步数据块的版本号和长度，上报结果给Namenode并进行数据块信息的更新。 

1）选择恢复主节点
首先在正常工作的数据流管道成员中选择一个作为恢复的主节点，其他节点作为参与节点，并将这些信息加入到主数据节点描述符中，等待该数据节点的心跳上报；

2）修改恢复文件的租约信息
名字节点触发的租约恢复会修改恢复文件的租约信息，他们的租约持有者统一改为NN_Recovery ；

3）租约恢复收集数据块信息
主数据节点收到指令后开始租约恢复，首先联系各个参与节点，开始租约恢复收集数据块信息.

4）找最小值数据块为恢复后的长度
根据收集到的数据块信息，找到最小值的教据块作为数据块恢复后的长度。

5）申请新数据块版本号
主数据节点向名字节点重新申请数据块版本号

6）同步长度和版本号
将长度和版本号同步到各个节点，同步结束后这些数据块拥有相同的大小和新的版本号。更新主要是利用DataNode上的FSDataSet.updata()方法进行更新

7）将恢复的结果上报给namenode.

8）Namenode更新块映射信息等
名字节点更新blockMap以及DataNodedescriptor中的信息等



**租约检测：** 

第1步，获取最老的已过期的租约。

 第2步，得到此租约中保存的文件Id。

 第3步，关闭这些文件Id对应的文件，并将这些文件Id从此租约中移除。 

第4步，如果此租约中已经没有打开的文件Id，则将此租约从系统中进行移除。 



## 10 HDFS中写数据的时候，写入副本出错怎么处理？

1、使用数据队列和确认队列来保证，无论那个节点故障了，都不会发生数据的丢失

2、使用租约恢复：

> 正常的数据节点辉被赋予一个新的版本号（租约信息中的时间戳版本），当故障恢复后发现版本信息不对，故障节点就会本删除
>
> 在当前正常的数据节点根据租约信息找到一个主DataNode，并与其他节点通信，选择虽小的数据块大小的DataNode，与其他节点进行同步，之后重新建立管道

3、NameNode的副本机制，在管线中删除故障节点，带文件关闭后，namenode发现副本不会，自然会进行复制副本。



## 11 HDFS删除一个文件的过程？

当数据块损坏、多余数据块、无效数据块等 Datanode会异步单独开启线程删除磁盘数据 

1） 客户端通过RPC，执行ClientProtocol.delete()发送删除请求

2 ）Namenode获得删除数据块的信息，执行底层的方法，删除所有数据块和对应的租约信息（先删除数据库信息和meta信息，删除之后执行数据节点中的数据块删除）

**注意：**
HDFS中的数据删除并不是直接删除，而是先放到类似于回收站的地方（trash），可供恢复。
当达到生命期限后（6小时），将彻底删除，并由Namenode修改相关的元数据信息
可以清空trash：bin/hadoop dfs expunge



## 12 Datanode启动、心跳已经执行名字节点指令流程

1）握手获取版本

Datanode启动，会通过DatanodeProtocol.versionRequest()获取Namenode版本以及存储信息，与Datanode的版本信息比较，确保一致 

2）注册

Datanode通过DatanodeProtocol.register()方法向Namenode注册，Namenode接受到注册请求后，判断版本号是否一致 

3）块汇报和缓存汇报

注册成功后，Datanode就需要将本地的所有数据块以及缓存数据块上报到Namenode，Namenode会利用这些信息重新建立内存中数据块与Datanode之间的对应关系 

4）发送心跳

上报之后数据节点通过sendHeartBeat方法与namenode通过心跳进行通信，Namenode会在给Datanode的心跳响应中携带名字节点指令，知道Datanode进行数据操作。 



## 13 HDFS通信协议

1）Hadoop RPC接口：六个接口

>客户端与名字节点的通信接口：ClientProtocol 
>
>客户端与数据节点的通信接口：ClientDataNodeProtocol 
>
>数据节点与名字节点通信接口：DatanodeProrocol(握手，注册，发送心跳，进行全量以及增量的数据块汇报)
>
> 数据节点与数据节点间的通信接口：InterDatanodeProtocol 
>
>第二名字节点与名字节点间的接口：NamenodeProtocol（2.X引入了HA机制，检查点操作也不由SecondNamenode来执行了） 
>
>其他接口：安全接口、HA接口 



2）流式接口

>基于TCP的Data TransferProtocol接口，来实现写入和读取数据（TCP利于大文件的批处理和高吞吐率） 
>
>基于Active Namenode和Standby Namenode间的HTTP接口 



## 14 HDFS2.X 联邦机制

**1）1.X的缺点导致引入了联邦机制** 

HDFSl.x架构使用一个Namenode来管理文件系统的命名空间以及数据块信息，这虽然使得HDFS的实现非常简单，但是单一的Namenode会导致以下缺点。

- 由于Namenode在内存中**保存整个文件系统的元数据**，所以Namenode内存的大小直接限制了文件系统的大小。
- 由于HDFS文件的读写等流程都涉及与Namenode交互，所以**文件系统的吞吐量受限于单个Namenode的处理能力。**
- Namenode作为文件系统的中心节点，**无法做到数据的有效隔离**。
- Namenode是集群中的单一故障点，有可用性隐患
- Namenode实现了数据块管理以及命名空间管理功能，造成这两个功能**高度耦合**，难以让其他服务单独使用数据块存储功能。



**2）考虑到上述缺点，为了能够水平扩展Namenode，HDFS2.X引入了联邦机制，提供了Federation架构：**



Federation架构的HDFS集群可以定义多个Namenode/Namespace，这些Namenode之间是相互独立的， '它们各自分工管理着自己的命名空间。 

![img](https://upload-images.jianshu.io/upload_images/5959612-f36bd1603e29b8bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/627/format/webp) 

**3）HDFS Federation两个新的概念：** 

**块池：** BlockPool 

一个块池由多个数据块组成，属于同一个命名空间下，这个块池中的数据块可以存储在集群中的所有数据节点上上，而每个Datanode都可以存储集群中所有块池的数据块。 每个块池是独立管理的，不会与其他的块池交互，所以一个Namenode出现故障时，并不会影响集群中的Datanode服务于其他的Namenode； 

**命名空间卷：**NamespaceVolume。 

一个Namenode管理的命名空间以及它对应的块池一起被称为命名空间卷，当一个Namenode/Namespace被删除后,它对应的块池也会从集群的数据节点上删除。需要特别注意的是，**当集群升级时，每个命名空间卷都会作为一个基本的单元进行升级** 



**4）联邦机制的优势**

支持Namenode/NameSpace的水平扩展，同时为用户和应用程序提供了命名空间卷级别的隔离性

联邦机制实现简单，NameNode不需要怎么改变，只需更改Datanode的部分代码即可。 例如将块池作为数据块存储的一个新层次，以及更改Datanode内部的数据结构等。 

可以分离Block storage层 ，解耦合命名空间管理和块存储管理；绕过Namenode/Namespace直接管理数据块，例如：Hbase可直接使用数据块； 可以在块存储上构建新文件系统（non-HDFS） 



## ----------------Yarn----------------

## 1 Yarn中的任务调度算法和队列

1）队列调度：FIFO Scheduler (默认)

>FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用进行分配资源，待最头上的应用需求满足后再给下一个分配，以此类推。
>
>缺点：FIFO Scheduler它并不适用于共享集群。**大的应用**可能会占用所有集群资源，这就导致其它应用被阻塞。

2）容量调度：Capacity Scheduler 

>Capacity调度器，有一个专门的队列用来运行小任务，但是为小任务专门设置一个队列**会预先占用一定的集群资源，**这就导致大任务的执行时间会落后于使用FIFO调度器时的时间。
>
>Capacity 调度器允许多个组织**共享整个集群，**每个组织可以获得集群的一部分计算能力。通过为每个组织分配专门的队列，然后再为每个队列分配一定的集群资源，这样整个集群就可以通过设置多个队列的方式给多个组织提供服务了。除此之外，队列内部又可以垂直划分，这样一个组织内部的多个成员就可以共享这个队列资源了，**在一个队列内部，资源的调度是采用的是先进先出(FIFO)策略。**
>
>当队列已满，Capacity调度器不会强制释放Container，当一个队列资源不够用时，这个队列只能获得其它队列释放后的Container资源，这个称为“弹性队列”，也可以设置最大值，防止过多占用其他队列的资源。



3）公平调度：Fair Scheduler 

>Fair调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。
>
>最终的效果就是Fair调度器即得到了高的资源利用率又能保证小任务及时完成。



## 2 Yarn的优势 

在Hadoop1时：

>1、JobTracker容易存在单点故障
>
>2、JobTracker负担重，既要负责资源管理，又要进行作业调度；当需处理太多任务时，会造成过多的资源消耗。
>
>3、当mapreduce job非常多的时候，会造成很大的内存开销，在 
>TaskTracker端，**以mapreduce task的数目作为资源的表示过于简单**，没有考虑到cpu以及内存的占用情况，如果两个大内存消耗的task被调度到了一块，很容易出现OutOfMemory异常。
>
>4、在TaskTracker端，把资源强制划分为`map task slot`和`reduce task slot`，如果当系统中只有map task或者只有reduce task的时候，会造成资源的浪费。

**解决单点故障和资源管理分配：**在Hadoop2使用Yarn来统一管理集群资源，Yarn把JobTracter分为ResourceManager和ApplactionMaster，ResouceManager专管整个集群的资源管理和调度，而ApplicationMaster则负责应用程序的任务调度和容错等 

**扩展性增强：**YARN不再是一个单纯的计算框架，而是一个框架管理器，用户可以将各种各样的计算框架移植到YARN之上，由YARN进行统一管理和资源分配

**资源管理单位：**对于资源的表示以内存和CPU为单位，比之前slot 更合理 



## 4 Yarn的架构

YARN主要由**ResouceManager、NodeManager、ApplicationMaster**和**Container**等4个组件构成 

**1）ResourceManager ：** 

处理客户端的请求、启动和管理ApplicationMaster、监控NameManager（心跳机制）、资源的分配和管理

**2）NodeManager：** 

管理节点上的资源、处理来自ResourceManager的命令、处理来自ApplicationMaster的命令

**3）ApplicationMaster：** 

为任务申请资源并分配给内部的任务、对任务进行监控和容错

**4）Container：** 

对任务的运行环境进行抽象，封装CPU内存等资源



## 5 MR的任务在Yarn上的运行过程

![img](https://upload-images.jianshu.io/upload_images/5959612-7831d525b3c24293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/598/format/webp) 

1）客户端向resourcemanager**发送job请求**，客户端产生Runjar进程与resourcemanager通过rpc通信 

2）resourcemanager向客户端**返回Job相关资源的提交路径**以以及**jobID** 

3）客户端将Job相关的资源提交到相应的共享文件系统的路径下（eg.HDFS） 

4）客户端向resourcemanager**提交job** 

5）resourcemanager通过调度器在nodemanager**创建一个容器**，**并在容器中启用 MRAppMaster进程**（该进程由resourcemanager启动）。 

6）该MRAppMaster进程**对作业进行初始化**，创建多个对象对作业进行跟踪。 

7）MRAppMaster从共享文件系统中**获得计算到的输入分片**（只获取分片信息，不需要jar等相关资源），为每一个分片**创建一个map以及指定数量的reduce对象**。之后 MRAppMaster决定如何运行构成mapreduce作业的各个任务，如果作业很小，则与 MRAppMaster在同一个JVM上运行 

8）若作业很大，则MRAppMaster会为所有map任务和reduce任务向resourcemanager**发起申请容器资源请求。**请求中包含了map任务的数据本地化信息以及输入分片等信息。 

9）Resourcemanager为任务分配了容器之后，MRAppMaster就通过与NodeManager通信启动容器，**由MRAppMaster负责分配在那些nodemanager上运行map和reduce任务。** 

10）运行map和reduce任务的nodemanager**从共享文件系统中获取job相关资源，包括 jar文件，配置文件**等。 

11）运行map和reduce任条。 

12）关于状态的检测与更新不经过resourcemanager：执行的任务周期性的向MRAppMaster汇报状态及进度，客户端每秒钟通过查询一次MRAppMaster获取状态更新信息。 

>1.由于resourcemanager负责资源的分配，当NodeManager启动时，会向 ResourceManager注册，而注册信息中会包含该节点可分配的CPU和内存总量 
>
>2.YARN的资源分配过程是异步的，也就是说，资源调度器将资源分配给一个应用后，不会立刻push给对应的ApplicaitonMaster，而是暂时放到一个缓冲区中，等待 ApplicationMaster通过周期性的RPC函数主动来取。 

## 6 Yarn的容错

**RM：**HA方案避免单点故障 

**AM：**AM向RM周期性发送心跳，出故障后RM会启动新的AM，受最大失败次数限制 

**NM：**周期性RM发送心跳，如果一定时间内没有发送，RM 就认为该NM 挂掉了，或者NM上运行的Task失败次数太多，就会把上面所有任务调度到其它NM上 

**Task：**Task也可能运行挂掉，比如内存超出了或者磁盘挂掉了，NM会汇报AM，AM会把该Task调度到其它节点上，但受到重试次数的限制 



## 7 资源调度和资源隔离机制

**资源调度**由ResourceManager完成；

**资源隔离**由各个NodeManager实现。

资源管理由ResourceManager和NodeManager共同完成，其中，ResourceManager中的调度器负责资源的分配，而NodeManager则负责资源的供给和隔离，ResourceManager将NodeManager上资源分配给任务后，NodeManager需按照要求为任务提供相应的资源，甚至保证这些资源应具有独占性，为任务运行提供基础的保证，这就是所谓的资源隔离。 



## 8 分布式架构的CAP原则

1）Consistency（一致性）：每次读操作都能保证返回的是最新数据；

2）Availability（可用性）：任何一个没有发生故障的节点，会在合理的时间内返回一个正常的结果；
备注：在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性），换句话就是说，任何时候，任何应用程序都可以读写数据。

3）Partition tolerance（分区容错性）：以当节点间出现网络分区，照样可以提供服务。

**CAP理论指出：CAP三者只能取其二，不可兼得。** 





