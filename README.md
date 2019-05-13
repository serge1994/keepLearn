# keepLearn
学习知识点

### 1、String StringBuffer 和 StringBuilder 的区别是什么 String 为什么是不可变的
简单的来说：String 类中使用 final 关键字字符数组保存字符串，private　final　char　value[]，所以 String 对象是不可变的。而StringBuilder 与 StringBuffer 都继承自 AbstractStringBuilder 类，在 AbstractStringBuilder 中也是使用字符数组保存字符串char[]value 但是没有用 final 关键字修饰，所以这两种对象都是可变的。

StringBuilder 与 StringBuffer 的构造方法都是调用父类构造方法也就是 AbstractStringBuilder 实现的，大家可以自行查阅源码。

AbstractStringBuilder.java
```
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    char[] value;
    int count;
    AbstractStringBuilder() {
    }
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
```
# Flink

## 数据库 ##

> 关系型数据库文章以MySQL为主

| 关系型数据库(MySQL/Oracle) | Redis | 
| :------:| :------: | 
| [关系型数据库(MySQL/Oracle)](src/database.md) | [Redis从零单排](src/redis.md) | 

## 大数据组件 ##

| HBASE优化 | spark调度模式 | MapReduce | 流计算 | kafka | 数仓调度系统 | kylin的调优 | carbondata |
| :------:| :------: | :------:| :------: | :------:| :------: | :------:| :------:|
| [HBASE优化](src/hbase.md) | [spark调度模式](src/spark.md) | [MapReduce](src/MR.md) | [流计算](src/real.md) | [kafka](src/kafka.md) | [数仓调度系统](src/数仓调度系统.md) | [kylin的调优](src/kylin.md) | [carbondata](src/carbondata.md) |

面试：
```
HBASE优化 
spark调度模式
MapReduce 
理论知识
kafka 流计算
数仓调度系统 及意义
```
 

