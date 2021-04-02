# Flink sql 

## 双流join

数栈ttl参数  ： sql.max.ttl=24h  sql.min.ttl=24h

会把所有的状态存储在磁盘上，每次触发任务会从最新的数据开始检索，数据量大会造成反压

左右流只要有新的数据进入就会触发左右ttl全量state数据join

## flink sql报错

```
Rowtime attributes must not be in the input rows of a regular join. As a workaround you can cast the time attributes of input tables to TIMESTAMP before.
```

参考：https://help.aliyun.com/knowledge_detail/113571.html

--------------------------------------------------------------------------------------------------------------------------------------

# [Batch (DataSet API)](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/batch/)

## Local Execution (本地测试debugging)

```xml
<!--        flink本地测试依赖-->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.11</artifactId>
  <version>1.8.0</version>
</dependency>
```

`ExecutionEnvironment.getExecutionEnvironment()` is the even better way to go then `ExecutionEnvironment.createLocalEnvironment()`. That method returns a `LocalEnvironment` when the program is started locally (outside the command line interface), and it returns a pre-configured environment for cluster execution, when the program is invoked by the [command line interface](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/cli.html).

**Flink provides a Command-Line Interface (CLI) to run programs that are packaged as JAR files**





# Flink DataStream API

## Overview

### 数据源

#### 未完成-自定义数据源

`addSource` - Attach a new source function. For example, to read from Apache Kafka you can use `addSource(new FlinkKafkaConsumer08<>(...))`. See [connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/connectors/index.html) for more details.

StreamExecutionEnvironment.addSource(sourceFunction)  —》自定义数据源

- 通过实现 `SourceFunction` 来写 non-parallel, 非并行 sources
- 通过实现 `ParallelSourceFunction` interface 或者继承 `RichParallelSourceFunction` 来写parallel sources.

#### 内置数据源

- `readTextFile(path)` - Reads text files, i.e. files that respect the `TextInputFormat` specification, line-by-line and returns them as Strings.
- `readFile(fileInputFormat, path)` - Reads (once) files as dictated by the specified file input format.
- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)`
- `socketTextStream` - Reads from a socket. Elements can be separated by a delimiter.
- `fromCollection(Collection)` - Creates a data stream from the Java Java.util.Collection. All elements in the collection must be of the same type.
- `fromCollection(Iterator, Class)` - Creates a data stream from an iterator. The class specifies the data type of the elements returned by the iterator.
- `fromElements(T ...)` - Creates a data stream from the given sequence of objects. All objects must be of the same type.
- `fromParallelCollection(SplittableIterator, Class)` - Creates a data stream from an iterator, in parallel. The class specifies the data type of the elements returned by the iterator.
- `generateSequence(from, to)` - Generates the sequence of numbers in the given interval, **in parallel.**



### 未完成-DataStream 算子转换

Please see [operators](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/operators/index.html) for an overview of the available stream transformations.

### 结果数据源

#### 内置sink

> 这些sink只支持at least-once语义，不参与checkpoint，一般用来测试。如果想用exact-once语义写入文件，可用flink-connector-filesystem来操作，或者通过自定义sink

- `writeAsText()` / `TextOutputFormat` - Writes elements line-wise as Strings. The Strings are obtained by calling the *toString()* method of each element.
- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field delimiters are configurable. The value for each field comes from the *toString()* method of the objects.
- `print()` / `printToErr()` - Prints the *toString()* value of each element on the standard out / standard error stream. Optionally, a prefix (msg) can be provided which is prepended to the output. This can help to distinguish between different calls to *print*. If the parallelism is greater than 1, the output will also be prepended with the identifier of the task which produced the output.
- `writeUsingOutputFormat()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports custom object-to-bytes conversion.
- `writeToSocket` - Writes elements to a socket according to a `SerializationSchema`

#### 未完成-自定义sink

`addSink` - Invokes a custom sink function. Flink comes bundled with connectors to other systems (such as Apache Kafka) that are implemented as sink functions.

### Iterations

以下程序从一系列整数中连续减去1，直到它们达到零为止：

```java
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
```



### 未完成-环境参数使用

参考 [execution configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/execution_configuration.html) 学习参数使用. These parameters pertain specifically to the DataStream API:

- `setAutoWatermarkInterval(long milliseconds)`: Set the interval for automatic watermark emission. You can get the current value with `long getAutoWatermarkInterval()`

#### 容错能力

[State & Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/state/checkpointing.html) describes how to enable and configure Flink’s checkpointing mechanism.

#### 控制延迟

通过`env.setBufferTimeout(timeoutMillis)`设置环境（或算子）的缓冲区填充的最大等待时间。在此时间之后，即使缓冲区未满，也会自动发送缓冲区。此超时的默认值为100毫秒。为了最大程度地提高吞吐量，请设置set `setBufferTimeout(-1)`来消除超时，并且仅在缓冲区已满时才刷新缓冲区。为了最大程度地减少延迟，请将超时设置为接近0的值（例如5或10 ms）。应避免将缓冲区超时设置为0，因为它可能导致严重的性能下降。

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

```java
// 源码
public StreamExecutionEnvironment setBufferTimeout(long timeoutMillis) {
        if (timeoutMillis < -1L) {
            throw new IllegalArgumentException("Timeout of buffer must be non-negative or -1");
        } else {
            this.bufferTimeout = timeoutMillis;
            return this;
        }
    }
```



### 本地调试

- [Local Execution Environment](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#local-execution-environment)
- [Collection Data Sources](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#collection-data-sources)
- [Iterator Data Sink](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#iterator-data-sink)

```java
// 使用DataStreamUtils.collect()可直接打印结果，可以不使用env.execute来执行
package com.dtstack.flink.local.apache.demo;

import com.dtstack.flink.local.EnvFactory;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.DataStreamUtils;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Iterator;

/**
 * @description: https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#debugging
 * @package: com.dtstack.flink.local.apache.demo
 * @author: liudang
 * @date: 2020-04-14 01:08
 **/
public class StreamLocalDebug {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment streamEnv = EnvFactory.getStreamEnv().setParallelism(1);
        DataStreamSource<Long> source = streamEnv.generateSequence(0, 20);// 并发生成

        SingleOutputStreamOperator<Tuple2<String, Long>> lisijie = source.map(new MapFunction<Long, Tuple2<String, Long>>() {
            @Override
            public Tuple2<String, Long> map(Long aLong) throws Exception {
                return new Tuple2<>("test", aLong);
            }
        });

        SingleOutputStreamOperator<Tuple2<String, Long>> sum = lisijie.keyBy(0).sum(1);

        Iterator<Tuple2<String, Long>> collect = DataStreamUtils.collect(sum);

        while (collect.hasNext()){
            Tuple2<String, Long> next = collect.next();
            System.out.println(next.toString());
        }
    }
}

```



# Event Time











## kafka connector使用

```xml
<!--        kafka 消费依赖-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_2.11</artifactId>
            <version>1.8.0</version>
        </dependency>
```

```bash
# 创建topic
sh kafka-topics.sh --create --zookeeper 172.16.100.188:2181,172.16.101.186:2181,172.16.101.224:2181/kafka --topic test0402 --partitions 1 --replication-factor 1

#mock数据到topic
./kafka-producer-perf-test.sh --producer-props bootstrap.servers=172.16.101.224:9092 --topic test0402      --num-records 10  --record-size 30 --throughput 5000
```

> 实时count topic数据

```java
package com.dtstack.flink.local;

import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.util.Collector;

import java.util.Properties;

/**
 * @description: 读取topic数据 ，打印
 * @package: com.dtstack.flink.local
 * @author: liudang
 * @date: 2020-04-13 15:01
 **/
public class readKafka {
    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "172.16.101.224:9092");
// only required for Kafka 0.8
        properties.setProperty("zookeeper.connect", "172.16.100.188:2181,172.16.101.186:2181,172.16.101.224:2181");
        properties.setProperty("group.id", "test1");
        FlinkKafkaConsumer<String> test0402 = new FlinkKafkaConsumer<>("test0402", new SimpleStringSchema(), properties);
        test0402.setStartFromEarliest();
        SingleOutputStreamOperator<Tuple2<String, Integer>> stream = env
                .addSource(test0402).filter(new FilterFunction<String>() {
                    @Override
                    public boolean filter(String s) throws Exception {
                        return s.endsWith("KR");
                    }
                }).flatMap(new FlatMapFunction<String, Tuple2<String,Integer>>() {
                    @Override
                    public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
                        collector.collect(new Tuple2<>(s,1));
                    }
                }).keyBy(0).sum(1);

        stream.print();

        env.execute();
    }
}

```

# Flink Table api













# 其他

##### 报错记录

> InvalidProgramException: Specifying keys via field positions is only valid for tuple data types
>
> 原因：**Java程序引用了Scala的Tuple2类**





##### 附录：stream api pom配置

```xml
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!--Flink 版本-->
        <flink.version>1.8.0</flink.version>
        <!--JDK 版本-->
        <java.version>1.8</java.version>
        <!--Scala 2.11 版本-->
        <scala.binary.version>2.11</scala.binary.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Apache Flink dependencies -->
        <!-- These dependencies are provided, because they should not be packaged into the JAR file. -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>


        <!-- Add logging framework, to produce console output when running in the IDE. -->
        <!-- These dependencies are excluded from the application JAR by default. -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.7</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
            <scope>runtime</scope>
        </dependency>

<!--        kafka 消费依赖-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_2.11</artifactId>
            <version>1.8.0</version>
        </dependency>
    </dependencies>

    <profiles>
        <profile>
            <id>add-dependencies-for-IDEA</id>

            <activation>
                <property>
                    <name>idea.version</name>
                </property>
            </activation>

            <dependencies>
                <dependency>
                    <groupId>org.apache.flink</groupId>
                    <artifactId>flink-java</artifactId>
                    <version>${flink.version}</version>
                    <scope>compile</scope>
                </dependency>
                <dependency>
                    <groupId>org.apache.flink</groupId>
                    <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
                    <version>${flink.version}</version>
                    <scope>compile</scope>
                </dependency>
            </dependencies>
        </profile>
    </profiles>
```





##### 数栈debug

线上环境风险很大。。。。DTapp/engine/flinkconf/log4j 调整日志级别debug







