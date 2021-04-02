



# 广播超时

org.apache.spark.SparkException: Exception thrown in awaitResult: 

## 数栈日志

``` 
====================基本日志====================
#log<info>log#2020-07-08 03:04:33:submit job is success
====================第 3次重试====================
====================LogInfo start====================
{"jobid":"application_1593691589966_6049","msg_info":"2020-07-08 02:56:39:submit job is success"}
=====================LogInfo end=====================
==================EngineInfo  start==================
{"driverLog": [], "appLog": [{"id":"application_1593691589966_6049","value":"User class threw exception: org.apache.spark.SparkException: Exception thrown in awaitResult: "}]}
===================EngineInfo  end===================
==================第3次重试结束==================
==
==
==
==
==
==
==
==
==
==
#log<info>log#   
 
#log<error>log##log<info>log# 
====================appLogs==================== 
application_1593691589966_6049 
User class threw exception: org.apache.spark.SparkException: Exception thrown in awaitResult: #log<info>log##log<error>log# 
```

## 两种方案：

https://blog.csdn.net/goodstudy168/article/details/83752989

https://mail-archives.apache.org/mod_mbox/spark-user/201509.mbox/%3CCAPn6-YSVRVSNkqHWHqARBx9PeC_qvJXitYZO0pC0xgaEeWqvfQ@mail.gmail.com%3E

```
配置的最大字节大小是用于当执行连接时，该表将广播到所有工作节点。通过将此值设置为-1，广播可以被禁用。
于是将此配置设整下，结果任务正常跑完。此处记录下，以示记忆。
sqlContext.setConf("spark.sql.autoBroadcastJoinThreshold", "-1")
目前加上spark.sql.autoBroadcastJoinThreshold=-1参数可成功运行任务

增大广播时间
You can change "spark.sql.broadcastTimeout" to increase the timeout. The
default value is 300 seconds.
```

# 小文件问题

数据量很小，但是文件超多，涉及多union all ，join等操作的sql任务，导致读取大量小、空文件，执行贼慢，加core并发，加资源也很难提升效率，同时任务时长过久，导致广播变量超时，所以关闭了上面的参数，稍微复杂的sql要跑几个小时

查数栈的小文件表

```shell
hadoop fs -count /dtInsight/hive/warehouse/aecc_mall.db/* |awk '$2>800 && $4!~/select/ {print $2,$3/1024/1024,$4}'

df -h|awk '{printf "%-15s %.2f\n",$1,$7/1024/1024}'
```



下面是中航金网的一张表，11M，20多万文件，集群没有文件数量限制

![image-20200708150351758](https://tva1.sinaimg.cn/large/007S8ZIlly1ggkkxhbovij31li0c6nif.jpg)

![img](https://tva1.sinaimg.cn/large/007S8ZIlly1ggkkw7etzcj31ne0u0h6s.jpg)

[spark自定义hadoop/hive参数](http://spark.apache.org/docs/latest/configuration.html#custom-hadoophive-configuration)

[官网Hive on Spark: 参数推荐设置](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)

> If your Spark application is interacting with Hadoop, Hive, or both, there are probably Hadoop/Hive configuration files in Spark’s classpath.
>
> Multiple running applications might require different Hadoop/Hive client side configurations. You can copy and modify `hdfs-site.xml`, `core-site.xml`, `yarn-site.xml`, `hive-site.xml` in Spark’s classpath for each application. In a Spark cluster running on YARN, these configuration files are set cluster-wide, and cannot safely be changed by the application.
>
> The better choice is to use spark hadoop properties in the form of `spark.hadoop.*`, and use spark hive properties in the form of `spark.hive.*`. For example, adding configuration “spark.hadoop.abc.def=xyz” represents adding hadoop property “abc.def=xyz”, and adding configuration “spark.hive.abc=xyz” represents adding hive property “hive.abc=xyz”. They can be considered as same as normal spark properties which can be set in `$SPARK_HOME/conf/spark-defaults.conf`
>
> In some cases, you may want to avoid hard-coding certain configurations in a `SparkConf`. For instance, Spark allows you to simply create an empty conf and set spark/spark hadoop/spark hive properties.

