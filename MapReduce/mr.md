shuffle的过程分析https://www.cnblogs.com/ahu-lichang/p/6665242.html

### 1、概述

#### A、MapReduce定义

MapReduce是面向大数据并行处理的计算模型、框架和平台，其资源调度由Yarn完成（hadoop2.0及以后版本），任务资源隐含了以下三层含义：

​    Mapreduce 是一个基本集群的高性能并行计算平台

​    MapReduce是一个并行计算与运行软件框架

​    Mapreduce是一个并行程序设计模型与方法。

MapReduce的资源管理平台采用YARN，它是一种新的Hadoop资源管理器，它是一个通用资源管理系统，可为上层应用提供统一的资源管理和调度，它的引入为集群在利用率、资源统一管理和数据分享等方面带来了巨大好处。

### 2、架构原理



![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5ktu2ovgj20ye0mxmzz.jpg)

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5ktt43n0j20ye0mxmzz.jpg)

#### A、MapReduce过程

​    直观上，我们可以这样想：把输入的数据（input）拆分为多个键值对（key-value对），每一个键值对分别调用Map进行平行处理，每个Map会产生多个新的键值对然后对Map阶段产生的数据进行排序、组合，最后以键值对的形式输出最终结果。

​    过程详解：首先在启动MapReduce之前，确保要处理的文件放在HDFS上面；客户端提交MapReduce程序的时候，应用程序会首先向ResourceManager申请资源，ResourceManager会创建对应的Job，其实应该是jobID（jobID形如：job_20152345321_0001）。job提交前客户端会对文件进行逻辑切片（split），MR框架默认将一个块（Block）作为一个分片，客户端应用可以重定义块与分片的映射关系。job提交给RM，RM根据NM的负载在NM集群中挑选合适的节点调度AM，AM负责job任务的初始化并向RM申请资源，有RM调度合适的NM启动Container，Container来执行Task。Map的输出会放入一个唤醒内存缓冲区，当缓冲区数据溢出时，需将缓冲区的数据写入到本地磁盘，写入本地磁盘之前通常需要做如下处理：1、分区（Partition）、2、排序（Sort）、3、组合（Combine）和4、合并（Spill）。

1、分区（Partition）分区时默认采用Hash算法分区，MR框架根据Reduce Task个数来确定分区个数，具备相同Key值的记录最终被送到形同的Reduce Task来处理；

2、排序（Sort）排序是将Map输出的记录排序，例如将（'Hi','1'）,('Hello','1'）重新排序为（'Hello','1'）,('Hi','1')；

3、组合（Combine）组合这个动作MR框架默认是可选的。例如将（'Hi','1'）,('Hi','1'),('Hello','1'),('Hello','1')进行合并操作为('Hi','2')，('Hi','2')；

4、合并（Spill）合并：Map Task在处理后会产生很多的溢出文件（spill file），这时需要将多个溢出文件进行合并处理，生成一个经过分区和排序的Spill File（MOF:MapOutFile）。为减少写入磁盘的数量，MR支持对MOF进行压缩后再写入。

通常在Map Task 任务完成MOF输出进度到3%时启动Reduce，从各个Map Task获取MOF文件。前面提到Reduce Task个数由客户端决定，Reduce Task个数决定MOF文件分区数。因此Map Task输出的MOF文件都能找到相对应的Reduce Task来处理。MOF文件是经过排序处理的。当Reduce Task接收的数据量不大时，则直接存放在内存缓冲区中，随着缓冲区文件的增多，MR后台线程将它们合并成一个更大的有序文件，这个动作是Reduce阶段的Merge操作，这个过程中会产生许多中间文件，最后一次合并的结果直接输出到用户自定义的reduce函数。

 Shuffle的定义：Map阶段和Reduce阶段之间传递数据的过程，包括Reduce Task从各个Map Task获取MOF文件的过程，以及对MOF的排序与合并处理。

 分片的必要性：MR框架将一个分片和一个Map Tast对应，即一个Map Task只负责处理一个数据分片。数据分片的数量确定了为这个Job创建Map Task的个数。

 Application Master(AM)负责一个Application生命周期内的所有工作。包括：与RM调度器协商以获取资源；将得到的资源进一步分配给内部任务（资源的二次分配）；与NM通信以启动/停止任务；监控所有任务运行状态，并在任务运行失败时重新为任务申请资源以重启任务。

 ResourceManager(RM) 负责集群中所有资源的统一管理和分配。它接收来自各个节点（NodeManager）的资源汇报信息，并根据收集的资源按照一定的策略分配给各个应用程序。

 NodeManager（NM）是每个节点上的代理，它管理Hadoop集群中单个计算节点，包括与ResourceManger保持通信，监督Container的生命周期管理，监控每个Container的资源使用（内存、CPU等）情况，追踪节点健康状况，管理日志和不同应用程序用到的附属服务（auxiliary service）。

Reduce阶段的三个过程：

 1.Copy---Reduce Task从各个Map Task拷贝MOF文件。

 2.Sort---通常又叫Merge，将多个MOF文件进行合并再排序。

 3.Reduce---用户自定义的Reduce逻辑。

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5ktvxrlqj20w90oon2p.jpg)