

![image-20191229141147643](/Users/admin/Library/Application Support/typora-user-images/image-20191229141147643.png)

# HIVE RegexSerDe使用详解

[原文]:https://blog.csdn.net/s530723542/article/details/38437257

通常情况下，hive导入的是单一分割符的数据。如果需要导入格式复杂一点的data，可以使用hive自导的RegexSerDe来实现。

RegexSerDe类是hive自带的，使用正则表达式来支持复杂的data导入。

在hive0.11中，自带了两个RegexSerDe类：

org.apache.hadoop.hive.contrib.serde2.RegexSerDe;

org.apache.hadoop.hive.serde2.RegexSerDe;

这两个类的区别在：

org.apache.hadoop.hive.serde2.RegexSerDe; 不支持output.format.string设定，设定了还会报警~~~~

org.apache.hadoop.hive.contrib.serde2.RegexSerDe;全部支持，功能比org.apache.hadoop.hive.serde2.RegexSerDe更强大，推荐使用org.apache.hadoop.hive.contrib.serde2.RegexSerDe。

下面对RegexSerDe类的介绍都是指：org.apache.hadoop.hive.contrib.serde2.RegexSerDe

1、使用方法：

示例：

	CREATE TABLE test_serde(  
			c0 string,  
			c1 string,  
			c2 string)  
			ROW FORMAT  
			SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'  
			WITH SERDEPROPERTIES  
			( 'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*)', 
			'input.regex.case.insensitive' = 'false'
			'output.format.string' = '%1$s %2$s %3$s')  
			STORED AS TEXTFILE; 
2、关键参数：
input.regex：输入的正则表达式
input.regex.case.insensitive：是否忽略字母大小写，默认为false
output.format.string：输出的正则表达式
3、注意事项：
a、使用RegexSerDe类时，所有的字段必须为string

b、input.regex里面，以一个匹配组，表示一个字段：([^ ]*)
————————————————

# hive特殊分隔符处理

补充： hive 读取数据的机制：
（1） 首先用 InputFormat<默认是： org.apache.hadoop.mapred.TextInputFormat >的一个具体实 现类读入文件数据，返回一条一条的记录（可以是行，或者是你逻辑中的“行”）
（2） 然后利用 SerDe<默认： org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe>的一个具体 实现类，对上面返回的一条一条的记录进行字段切割
Hive 对文件中字段的分隔符默认情况下只支持单字节分隔符，如果数据文件中的分隔符是多字符的，如下所示：
01||huangbo
02||xuzheng
03||wangbaoqiang
可用以下方式处理：

## 1、案例：使用 RegexSerDe 通过正则表达式来抽取字段

create table t_bi_reg(id string,name string)
row format serde 'org.apache.hadoop.hive.serde2.RegexSerDe'
with serdeproperties('input.regex'='(.*)\\|\\|(.*)','output.format.string'='%1$s %2$s')               如果三个字段，再加上一个%3$s
stored as textfile;

hive>load data local inpath '/root/hivedata/bi.dat' into table t_bi_reg;
hive>select * from t_bi_reg;

## 2、案例：Hive之——RegexSerDe来处理标准格式Apache Web日志

https://blog.csdn.net/l1028386804/article/details/88617592



# hive beeline 连接操作

- Spark thrift(em里spark thriftserver) : 默认!connect jdbc:hive2://xxx.xx.xx.xx:8191

```
hive.server2.thrift.port 值为准确端口号，可在em里查找thriftserver服务的hive-site配置文件
```

> Error: Could not open client transport with JDBC Uri: jdbc:hive2://172.16.101.237:10000: Could not establish connection to jdbc:hive2://172.16.101.237:10000: Required field 'client_protocol' is unset! Struct:TOpenSessionReq(client_protocol:null, configuration:{use:database=default}) (state=08S01,code=0)
>
> 若连接报错，是 hive-jdbc的version 和hive Server的version不一致，切换使用opt/dtstack/Hadoop/thriftserver/bin/beeline客户端。Driver: Hive JDBC (version 1.2.1.spark2)

- em里Hive metastore的hive-site配置文件中(hive.server2.thrift.port端口) :默认 !connect jdbc:hive2://172.16.100.119:10004

> /opt/dtstack/Hadoop/hivemetastore/bin/beeline客户端。Beeline version 2.1.1 by Apache Hive

- beeline 退出

```shell
beeline> !quit 
[root@app1 bin]# 
```

# hql、sparksql指定location建表的区别

Hive EXTERNAL子句创建的表称为[托管表](https://cwiki.apache.org/confluence/display/Hive/Managed+vs.+External+Tables)，由Hive管理其数据。要确定表是托管表还是外部表，使用DESCRIBE EXTENDED table_name的输出中查找tableType 

同时如果在外部表设置了4.0.0+（[HIVE-19981](https://issues.apache.org/jira/browse/HIVE-19981)）版本中的TBLPROPERTIES（“ external.table.purge” =“ true”），也会删除该数据。

```
托管表
托管表存储在hive.metastore.warehouse.dir路径属性下，默认情况下存储在类似于的文件夹路径中/user/hive/warehouse/databasename.db/tablename/。location在表创建期间，该属性可以覆盖默认位置。如果删除了托管表或分区，则将删除与该表或分区关联的数据和元数据。如果未指定PURGE选项，则数据将在定义的持续时间内移至回收站。
如果需要通过Hive管理表的生命周期或创建临时表时，使用托管表管理数据

外部表
外部表描述了外部路径文件上的metadata/schema。外部表文件可以由Hive外部的进程访问和管理。外部表可以访问存储在诸如Azure Storage Volumes（ASV）或远程HDFS位置的源中的数据。如果更改了外部表的结构或分区，则可以使用MSCK REPAIR TABLE table_name语句刷新元数据信息。

当文件已经存在或位于远程位置时，可使用外部表，并且即使表已删除，文件也会保留。
```

目前数栈使用版本为2.1.3，sparksql建表类型为external（[原因](https://stackoverflow.com/questions/44279412/how-to-create-external-hive-table-without-location):The default `SparkSqlParser` (with `astBuilder` as `SparkSqlAstBuilder`) has the [following assertion](https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkSqlParser.scala#L1095-L1096) that leads to the exception:）

hql表类型为managed，但通过hql建表元数据目前无法通过数栈管理，通过引入TBLPROPERTIES ("external.table.purge"="true") 配置，将外部表转为严格托管模式待测试

> 题外话：2.1.3版本，sparksql中不支持create external table外部表的创建，只能是非external表。使用write.option(“path”,"/some/path").saveAsTable是external表。
> 使用外部表，可以直接加载数据并加载到DateSet.createOrReplaceTempView中完成。
> 如果注册的表是createGlobalTempView，那么访问表需要加上数据库名，global_temp.tableName否在默认在default中查找会导致报错： Table or view ‘tableName’ not found in database ‘default’;
> ————————————————
> 原文链接：https://blog.csdn.net/mar_ljh/article/details/85210944
>
> CREATE EXTERNAL TABLE already works  https://issues.apache.org/jira/browse/SPARK-2825



# hive truncate table知识点

Truncate table partition数据会删除，hdfs文件夹(分区)不会删除

alter table drop  partition 删除数据文件且删除metadata。