# HDFS基本原理

## 1、概述

HDFS(Hadoop Distributed File System):Hadoop分布式文件系统。是分布式计算中数据存储管理的基础，是基于流数据模式访问和处理超大文件的需求而开发的。

## 2、HDFS特点

### A、优势

a、容错机制，会保存多个副本（3）

b、运行在廉价的机器上

c、适合大数据处理，处理数据达到GB、TB、甚至PB级别的数据；能够处理百万规模以上的文件数量，数量相当之大；能够吃力10k节点的规模；处理时会默认将文件分割成block，128M为1个block。然后将block按键值对存储在HDFS上，并将键值对的映射存到内存中。（所以如果小文件太多，内存负担会很重)；

d、适合批处理：它是通过移动计算而不是移动数据，它会把数据位置暴露给计算框架。

e、流式问及那访问，一次写入，多次读取，文件一点写入不能修改，只能追加，它能摆正数据的一致性。

### B、劣势--不适合的场合

a、低延时数据访问：它适合高吞吐率的场景，就是在某一时间内写入大量的数据。但是它在低延时的情况下是不行的，比如毫秒级以内读取数据，这样它是很难做到的。

b、小文件存储：存储大量小文件的话，它会占用 NameNode大量的内存来存储文件、目录和块信息。这样是不可取的，因为NameNode的内存总是有限的。小文件存储的寻道时间会超过读取时间，它违反了HDFS的设计目标。（这里的小文件是指小于HDFS系统的Block大小的文件，默认是128M）

c、并发写入、文件随机修改：一个文件只能有一个线程写，不允许多个线程同时写，且仅支持数据append（追加），不支持文件的随机修改。

## 3、存储结构

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5kyte7d9j20hf0arwlv.jpg)

正如上图，HDFS也是按照Master和Slave的架构来存储数据。分HDFS Client、NameNode、SecondaryNameNode、DataNode这几个角色。

### A、Client（客户端）

​        1、 切分文件，文件上传HDFS的时候，Client将文件切分成一个一个的Block，然后进行存储

​        2、与NameNode交互，获取文件的位置信息。

​        3、与DataNode交互，读取或者写入数据。

​        4、Client提供一些命令来管理HDFS，比如启动或者关闭HDFS。

​        5、Client可以通过一些命令来访问HDFS。

### B、NameNode

是Master节点上的进程，是大领导。管理数据块映射；处理客户端的读写请求；配置副本策略；管理HDFS的命名空间；NameNode提供的是始终被动接收服务的server。

### C、SecondaryNameNode

是一个小弟，分担大哥namenode的工作量；是NameNode的冷备份；定期合并fsimage和fsedits然后再发给namenode。

### D、DataNode

Slave节点上的进程，奴隶，干活的。负责存储client发来的数据块block；执行数据块的读写操作。

### E、热备份

b是a的热备份，如果a坏掉。那么b马上代替a工作。

### F、冷备份

b是a的冷备份，如果a坏掉。那么b不能马上代替a工作。但是b上存储a的一些信息，减少a坏掉之后的损失。

### G、fsimgage

元数据镜像文件（文件系统的目录树）

### H、edits

元数据的操作日志（针对文件系统做的修改记录）

**namenode**内存中存储的是=fsimage+edits。

**SecondaryNameNode**负责定时（默认1小时），从namenode上，获取fsimage和edits来进行合并，然后再发送给namenode。减少namenode的工作量。

### I、HDFS文件数据块副本存放策略

​    Hadoop 0.17之前的副本策略

​            第一个副本：存储在同机架的不同节点上。

​            第二个副本：存储在同机架的另外一个节点上。

​            第三个副本：存储在不同机架的另外一个节点。

​               其它副本：选择随机存储。

​    Hadoop 0.17 之后的副本策略

​            第一个副本：存储在同 Client 相同节点上。

​            第二个副本：存储在不同机架的节点上。

​            第三个副本：存储在第二个副本机架中的另外一个节点上。

​               其它副本：选择随机存储。

## 4、工作原理

### A、写操作

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5kyuv5iaj20tg0jlk8m.jpg)

有一个文件FileA，200M大小。Client将FileA写入到HDFS上。

HDFS按默认配置。

HDFS分布在三个机架上Rack1，Rack2，Rack3。

**a.** Client将FileA按128M分块。分成两块，block1和Block2;

**b.** Client向nameNode发送写数据请求，如图蓝色虚线①------>。

**c.** NameNode节点，记录block信息。并返回可用的DataNode，如粉色虚线②--------->。

​    Block1: host2,host1,host3

​    Block2: host7,host8,host4

​    原理：

​        NameNode具有RackAware机架感知功能，这个可以配置。

​        若client为DataNode节点，那存储block时，规则为：副本1，同client的节点上；副本2，不同机架节点上；副本3，同第二个副本机架的另一个节点上；其他副本随机挑选。

​        若client不为DataNode节点，那存储block时，规则为：副本1，随机选择一个节点上；副本2，不同副本1，机架上；副本3，同副本2相同的另一个节点上；其他副本随机挑选。

**d.** client向DataNode发送block1；发送过程是以流式写入。

​    流式写入过程，

​        **1>**将128M的block1按64k的package划分;

​        **2>**然后将第一个package发送给host2;

​        **3>**host2接收完后，将第一个package发送给host1，同时client想host2发送第二个package；

​        **4>**host1接收完第一个package后，发送给host3，同时接收host2发来的第二个package。

​        **5>**以此类推，如图红线实线所示，直到将block1发送完毕。

​        **6>**host2,host1,host3向NameNode，host2向Client发送通知，说“消息发送完了”。如图粉红颜色实线所示。

​        **7>**client收到host2发来的消息后，向namenode发送消息，说我写完了。这样就真完成了。如图黄色粗实线

​        **8>**发送完block1后，再向host7，host8，host4发送block2，如图蓝色实线所示。

​        **9>**发送完block2后，host7,host8,host4向NameNode，host7向Client发送通知，如图浅绿色实线所示。

​        **10>**client向NameNode发送消息，说我写完了，如图黄色粗实线。。。这样就完毕了。

**分析：**通过写过程，我们可以了解到：

​    **①**写1T文件，我们需要3T的存储，3T的网络流量带宽。

​    **②**在执行读或写的过程中，NameNode和DataNode通过HeartBeat进行保存通信，确定DataNode活着。如果发现DataNode死掉了，就将死掉的DataNode上的数据，放到其他节点去。读取时，要读其他节点去。

​    **③**挂掉一个节点，没关系，还有其他节点可以备份；甚至，挂掉某一个机架，也没关系；其他机架上，也有备份。

写入原理--具体流程

​       1.客户端通过调用DistributedFileSystem的create方法，创建一个新的文件。

　　2.DistributedFileSystem通过RPC（远程过程调用）调用NameNode，去创建一个没有blocks关联的新文件。创建前，NameNode 会做各种校验，比如文件是否存在，客户端有无权限去创建等。如果校验通过，NameNode就会记录下新文件，否则就会抛出IO异常。

　　3.前两步结束后会返回 FSDataOutputStream 的对象，和读文件的时候相似，FSDataOutputStream 被封装成 DFSOutputStream，DFSOutputStream 可以协调 NameNode和 DataNode。客户端开始写数据到DFSOutputStream,DFSOutputStream会把数据切成一个个小packet，然后排成队列 data queue。

　　4.DataStreamer 会去处理接受 data queue，它先问询 NameNode 这个新的 block 最适合存储的在哪几个DataNode里，比如重复数是3，那么就找到3个最适合的 DataNode，把它们排成一个 pipeline。DataStreamer 把 packet 按队列输出到管道的第一个 DataNode 中，第一个 DataNode又把 packet 输出到第二个 DataNode 中，以此类推。

　　5.DFSOutputStream 还有一个队列叫 ack queue，也是由 packet 组成，等待DataNode的收到响应，当pipeline中的所有DataNode都表示已经收到的时候，这时akc queue才会把对应的packet包移除掉。

　　6.客户端完成写数据后，调用close方法关闭写入流。

　　7.DataStreamer 把剩余的包都刷到 pipeline 里，然后等待 ack 信息，收到最后一个 ack 后，通知 DataNode 把文件标示为已完成。

### B、读操作

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1go5kyxob61j20t40jnwp4.jpg)

读操作就简单一些了，如图所示，client要从datanode上，读取FileA。而FileA由block1和block2组成。 

那么，读操作流程为：

**a.** client向namenode发送读请求。

**b.** namenode查看Metadata信息，返回fileA的block的位置。

​    block1:host2,host1,host3

​    block2:host7,host8,host4

**c.** block的位置是有先后顺序的，先读block1，再读block2。而且block1去host2上读取；然后block2，去host7上读取；

上面例子中，client位于机架外，那么如果client位于机架内某个DataNode上，例如,client是host6。那么读取的时候，遵循的规律是：

**优选读取本机架上的数据**。

读取原理--具体步骤：

1、首先调用FileSystem对象的open方法，其实获取的是一个DistributedFileSystem的实例。

2、DistributedFileSystem通过RPC(远程过程调用)获得文件的第一批block的locations，同一block按照重复数会返回多个locations，这些locations按照hadoop拓扑结构排序，距离客户端近的排在前面。

3、前两步会返回一个FSDataInputStream对象，该对象会被封装成DFSInputStream对象，DFSInputStream可以方便的管理datanode和namenode数据流。客户端调用read方法，DFSInputStream就会找出离客户端最近的datanode并连接datanode。

4、数据从datanode源源不断的流向客户端。

5、如果第一个block块的数据读完了，就会关闭指向第一个block块的datanode连接，接着读取下一个block块。这些操作对客户端来说是透明的，从客户端的角度来看只是读一个持续不断的流。

6、如果第一批block都读完了，DFSInputStream就会去namenode拿下一批blocks的location，然后继续读，如果所有的block块都读完，这时就会关闭掉所有的流。