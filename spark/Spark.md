# Spark

> 本地运行模式 （单机）
>
> - 　　该模式被称为Local[N]模式，是用单机的多个线程来模拟Spark分布式计算，直接运行在本地，便于调试，通常用来验证开发出来的应用程序逻辑上有没有问题。
> - 　　其中N代表可以使用N个线程，每个线程拥有一个core。如果不指定N，则默认是1个线程（该线程有1个core）。
> - 　　如果是local[*]，则代表 Run Spark locally with as many worker threads as logical cores on your machine.

```java
public void runJob(String submitSql, String appName, String logLevel, SparkConf conf){

    if(appName == null){
        appName = DEFAULT_APP_NAME;
    }

    SparkSession spark = SparkSession
            .builder()
            .config(conf)
            .appName(appName)
            .enableHiveSupport()
            .getOrCreate();

    setLogLevel(spark, logLevel);
    //解压sql
    String unzipSql = ZipUtil.unzip(submitSql);

    //屏蔽引号内的 分号
    List<String> sqlArray = DtStringUtil.splitIgnoreQuota(unzipSql, ';');
    for(String sql : sqlArray){
        if(sql == null || sql.trim().length() == 0){
            continue;
        }
        logger.info("processed sql statement {}", sql);
        spark.sql(sql);
    }

    spark.close();
}
```



### 错误日志

### 1flinksql

Caused by: org.apache.flink.client.program.ProgramInvocationException: The main method caused an error: SQL validation failed. From line 3, column 114 to line 3, column 168: Illegal mixing of types in CASE or COALESCE statement

```sql
INSERT    
INTO
    xyz_uuid_uv
    select
        uuid,
        sysdate,
        uc+uv                  
    from
        (SELECT
            t.uuid as uuid,
            t.sysdate as sysdate,
            count(t.uuid,
            t.sysdate) as uc,
            case                                       
                when v.uv is null then 0                                       
                else v.uv                               
            end as uv                           
        from
            (select
                uuid,
                cast (DATE_FORMAT(sysTime,
                '%Y%m%d') as varchar)as sysdate                                                
            from
                en_nginx_xyz_cmcc)t                                
        LEFT JOIN
            dim_xyz_uuid_uv v                                                                
                on (
                    t.uuid=v.uuid                                                                                
                    and t.sysdate=v.sysdate                                                                
                )                             
        group by
            t.uuid,
            t.sysdate,
            v.uv)tmp
```

```
## 平台支持hivesql，但poc环境没有新建hiveSql任务选项
客户疑问：从哪个版本开始支持hivesql和impalasql
## 上游任务重跑成功了，我再跑这个任务的时候还是显示失败
## 实时流支持项目 迁移吗？
流计算这边现在还不支持绑定发布，需要手动空间隔离，从开发环境复制代码到生产环境
## 客户关注jar开发的实时流是否能进行续跑
目前不支持，研发这边看下怎么去做
## mr任务运行任然报错
1.提交任务参数中的"typeName":"hadoop"
2.mr任务的参数提交后可以保存了，但保存任务后重新打开，参数回写到页面会丢失
mr代码里指定了mapper，reducer，但还是要传给main方法三个参数
## shell任务可以执行hadoop或者是hive的shell脚本吗？试了不行
## 能否支持公共资源，不同用户上传的资源作为公共资源进行共享共用，公共资源只需上传一次即可。
## 告警不生效的问题未解决，青火还在排查
```

- [客户疑问：](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#p3HVz)

- [能否支持公共资源，不同用户上传的资源作为公共资源进行共享共用，公共资源只需上传一次即可。](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#9458700d)

- [shell任务可以执行hadoop或者是hive的shell脚本吗？](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#571e84a8)

- [是否支持hivesql和impalasql](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#Ro6nk)

- [实时流支持项目迁移吗？](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#d1fd6743)

- [jar开发的实时流是否能进行续跑](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#600ecad4)

- [mr任务运行任然报错：](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#8f816e1c)

  [1.提交任务参数中的"typeName":"hadoop"](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#ZG4mq)

  [2.mr任务的参数保存任务后重新打开，参数回写到页面会丢失](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#C4QC1)

- [mr任务传自定义参数](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#qCaJY)

- [告警不生效的问题未解决，青火持续查看](https://dtstack.yuque.com/dtsupport/kc538g/eq54ss#00a8cc75)





![image-20200721201718461](https://tva1.sinaimg.cn/large/007S8ZIlly1ggyuyhivo0j31g50u0jwq.jpg)![image-20200721202014659](https://tva1.sinaimg.cn/large/007S8ZIlly1ggyv0kjq03j31g50u01ir.jpg)spark 逻辑计划优化，RBO？基于规则优化

物理计划优化 CBO,选择最优的物理计划,基于成本优化

核心catalyst模块:

parse:解析,生成逻辑计划

analyzer:语法解析sqlbaseparse.java,词法解析sqlbaselexer.java

optimizer:RBO 谓词下推，列剪枝，常量替换，常量计算

![image-20200721205748716](https://tva1.sinaimg.cn/large/007S8ZIlly1ggywdd5ukrj31hf0u07c3.jpg)

planner:CBO.   Spark2.3以后引入



其他：

Reducebykey  -》先在map端join

spark的sqlbase.g4  可查所有支持的sql语法