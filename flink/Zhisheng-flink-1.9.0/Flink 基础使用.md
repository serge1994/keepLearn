![img](https://tva1.sinaimg.cn/large/0081Kckwly1glvqh9441sj31oq0u0wt5.jpg)

# 7 Flink 中使用 Lambda 排坑

 Flink 中是支持 Lambda，但是不太友好。比如上面的应用程序如果将 LineSplitter 该类之间用 Lambda 表达式完成的话则要像下面这样写：

```java
stream.flatMap((s, collector) -> {
    for (String token : s.toLowerCase().split("\\W+")) {
        if (token.length() > 0) {
            collector.collect(new Tuple2<String, Integer>(token, 1));
        }
    }
})
        .keyBy(0)
        .sum(1)
        .print();
```

但是这样写完后，运行作业报错如下：

```
Exception in thread "main" org.apache.flink.api.common.functions.InvalidTypesException: The return type of function 'main(LambdaMain.java:34)' could not be determined automatically, due to type erasure. You can give type information hints by using the returns(...) method on the result of the transformation call, or by letting your function implement the 'ResultTypeQueryable' interface.
    at org.apache.flink.api.dag.Transformation.getOutputType(Transformation.java:417)
    at org.apache.flink.streaming.api.datastream.DataStream.getType(DataStream.java:175)
    at org.apache.flink.streaming.api.datastream.DataStream.keyBy(DataStream.java:318)
    at com.zhisheng.examples.streaming.socket.LambdaMain.main(LambdaMain.java:41)
Caused by: org.apache.flink.api.common.functions.InvalidTypesException: The generic type parameters of 'Collector' are missing. In many cases lambda methods don't provide enough information for automatic type extraction when Java generics are involved. An easy workaround is to use an (anonymous) class instead that implements the 'org.apache.flink.api.common.functions.FlatMapFunction' interface. Otherwise the type has to be specified explicitly using type information.
    at org.apache.flink.api.java.typeutils.TypeExtractionUtils.validateLambdaType(TypeExtractionUtils.java:350)
    at org.apache.flink.api.java.typeutils.TypeExtractionUtils.extractTypeFromLambda(TypeExtractionUtils.java:176)
    at org.apache.flink.api.java.typeutils.TypeExtractor.getUnaryOperatorReturnType(TypeExtractor.java:571)
    at org.apache.flink.api.java.typeutils.TypeExtractor.getFlatMapReturnTypes(TypeExtractor.java:196)
    at org.apache.flink.streaming.api.datastream.DataStream.flatMap(DataStream.java:611)
    at com.zhisheng.examples.streaming.socket.LambdaMain.main(LambdaMain.java:34)
```

根据上面的报错信息其实可以知道要怎么解决了，该错误是因为 Flink 在用户自定义的函数中会使用泛型来创建 serializer，当使用匿名函数时，类型信息会被保留。但 Lambda 表达式并不是匿名函数，所以 javac 编译的时候并不会把泛型保存到 class 文件里。

解决方法：使用 Flink 提供的 returns 方法来指定 flatMap 的返回类型

```java
//使用 TupleTypeInfo 来指定 Tuple 的参数类型
.returns((TypeInformation) TupleTypeInfo.getBasicTupleTypeInfo(String.class, Integer.class))
```

在 flatMap 后面加上上面这个 returns 就行了，但是如果算子多了的话，每个都去加一个 returns，其实会很痛苦的，所以通常使用匿名函数或者自定义函数居多。

# 9 Flink Window 基础概念与实现原理

## 窗口概念

**窗口通过周期性的规则(基于时间、数量等)将无界数据量逻辑划分为一批批的有界数据，可以认为窗口是流到批的一个桥梁**

## 窗口的生命周期

通过以上的内容，我们应该知道了窗口的作用(主要是为了解决什么样的问题)。那么这个时候需要思考四个问题

1. 数据元素是如何分配到对应窗口中的(也就是窗口的分配器)？
2. 元素分配到对应窗口之后什么时候会触发计算(也就是窗口的触发器)？
3. 在窗口内我们能够进行什么样的操作(也就是窗口内的操作)？
4. 当窗口过期后是如何处理的(也就是窗口的销毁关闭)？

其实这四个问题从大体上可以理解为窗口的整个生命周期过程。接下来我们对每个环节进行讲解

## 自带三种window

TimeWindow、CountWindow、SessionWindow

### Time Window 使用及源码分析

在 Flink 中使用 Time Window 非常简单，输入一个时间参数，这个时间参数可以利用 Time 这个类来控制，如果事前没指定 TimeCharacteristic 类型的话，则默认使用的是 ProcessingTime.

```java
dataStream.keyBy(1)
    .timeWindow(Time.minutes(1)) //time Window 每分钟统计一次数量和
    .sum(1);
```

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-15-155127.jpg)

该 timeWindow 方法在 KeyedStream 中对应的源码如下：

```java
//时间窗口
public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size) {
    if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
        return window(TumblingProcessingTimeWindows.of(size));
    } else {
        return window(TumblingEventTimeWindows.of(size));
    }
}
```

另外在 Time Window 中还支持滑动的时间窗口，比如定义了一个每 30s 滑动一次的 1 分钟时间窗口，它会每隔 30s 去统计过去一分钟窗口内的数据，同样使用也很简单，输入两个时间参数，如下：

```java
dataStream.keyBy(1)
    .timeWindow(Time.minutes(1), Time.seconds(30)) //sliding time Window 每隔 30s 统计过去一分钟的数量和
    .sum(1);
```

滑动时间窗口的数据聚合流程如下图所示：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-15-155522.jpg)

在该第一个时间窗口中（1 ～ 2 分钟）和为 7，第二个时间窗口中（1:30 ~ 2:30）和为 10，第三个时间窗口中（2 ~ 3 分钟）和为 12，第四个时间窗口中（2:30 ~ 3:30）和为 10，第五个时间窗口中（3 ~ 4 分钟）和为 7，第六个时间窗口中（3:30 ~ 4:30）和为 11，第七个时间窗口中（4 ~ 5 分钟）和为 19。

该 timeWindow 方法在 KeyedStream 中对应的源码如下：

```java
//滑动时间窗口
public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size, Time slide) {
    if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
        return window(SlidingProcessingTimeWindows.of(size, slide));
    } else {
        return window(SlidingEventTimeWindows.of(size, slide));
    }
}
```

### Count Window 使用及源码分析

Apache Flink 还提供计数窗口功能，如果计数窗口的值设置的为 3 ，那么将会在窗口中收集 3 个事件，并在添加第 3 个元素时才会计算窗口中所有事件的值。

在 Flink 中使用 Count Window 非常简单，输入一个 long 类型的参数，这个参数代表窗口中事件的数量，使用如下：

```java
dataStream.keyBy(1)
    .countWindow(3) //统计每 3 个元素的数量之和
    .sum(1);
```

计数窗口的数据窗口聚合流程如下图所示：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-16-045758.jpg)

该 countWindow 方法在 KeyedStream 中对应的源码如下：

```java
//计数窗口
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size) {
    return window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(size)));
}
```

另外在 Count Window 中还支持滑动的计数窗口，比如定义了一个每 3 个事件滑动一次的 4 个事件的计数窗口，它会每隔 3 个事件去统计过去 4 个事件计数窗口内的数据，使用也很简单，输入两个 long 类型的参数，如下：

```java
dataStream.keyBy(1) 
    .countWindow(4, 3) //每隔 3 个元素统计过去 4 个元素的数量之和
    .sum(1);
```

滑动计数窗口的数据窗口聚合流程如下图所示：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-16-065833.jpg)

该 countWindow 方法在 KeyedStream 中对应的源码如下：

```java
//滑动计数窗口
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
    return window(GlobalWindows.create()).evictor(CountEvictor.of(size)).trigger(CountTrigger.of(slide));
}
```

### Session Window 使用及源码分析

Apache Flink 还提供了会话窗口，是什么意思呢？使用该窗口的时候你可以传入一个时间参数（表示某种数据维持的会话持续时长），如果超过这个时间，就代表着超出会话时长。

在 Flink 中使用 Session Window 非常简单，你该使用 Flink KeyedStream 中的 window 方法，然后使用 ProcessingTimeSessionWindows.withGap()（不一定就是只使用这个），在该方法里面你需要做的是传入一个时间参数，如下：

```java
dataStream.keyBy(1)
    .window(ProcessingTimeSessionWindows.withGap(Time.seconds(5)))//表示如果 5s 内没出现数据则认为超出会话时长，然后计算这个窗口的和
    .sum(1);
```

会话窗口的数据窗口聚合流程如下图所示：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-16-150258.jpg)

该 Window 方法在 KeyedStream 中对应的源码如下：

```java
//提供自定义 Window
public <W extends Window> WindowedStream<T, KEY, W> window(WindowAssigner<? super T, W> assigner) {
    return new WindowedStream<>(this, assigner);
}
```

## 如何自定义 Window？

三个概念：WindowAssigner(窗口分配器)、Trigger(触发器)、Evictor(剔除器)

当然除了上面几种自带的 Window 外，Apache Flink 还提供了用户可自定义的 Window，那么该如何操作呢？其实细心的同学可能已经发现了上面我写的每种 Window 的实现方式了，它们有 assigner、 evictor、trigger。如果你没发现的话，也不要紧，这里我们就来了解一下实现 Window 的机制，这样我们才能够更好的学会如何自定义 Window。

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-23-073301.png)

### Window 源码定义

上面说了 Flink 中自带的 Window，主要利用了 KeyedStream 的 API 来实现，我们这里来看下 Window 的源码定义如下：

```java
public abstract class Window {
    //获取属于此窗口的最大时间戳
    public abstract long maxTimestamp();
}
```

查看源码可以看见 Window 这个抽象类有如下实现类：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-17-163050.png)

**TimeWindow** 源码定义如下:

```java
public class TimeWindow extends Window {
    //窗口开始时间
    private final long start;
    //窗口结束时间
    private final long end;
}
```

**GlobalWindow** 源码定义如下：

```java
public class GlobalWindow extends Window {

    private static final GlobalWindow INSTANCE = new GlobalWindow();

    private GlobalWindow() { }
    //对外提供 get() 方法返回 GlobalWindow 实例，并且是个全局单例
    public static GlobalWindow get() {
        return INSTANCE;
    }
}
```

### Window 组件之 WindowAssigner 使用及源码分析

到达窗口操作符的元素被传递给 WindowAssigner。WindowAssigner 将元素分配给一个或多个窗口，可能会创建新的窗口。

窗口本身只是元素列表的标识符，它可能提供一些可选的元信息，例如 TimeWindow 中的开始和结束时间。注意，元素可以被添加到多个窗口，这也意味着一个元素可以同时在多个窗口存在。我们来看下 WindowAssigner 的代码的定义吧：

```java
public abstract class WindowAssigner<T, W extends Window> implements Serializable {
    //分配数据到窗口并返回窗口集合
    public abstract Collection<W> assignWindows(T element, long timestamp, WindowAssignerContext context);
}
```

查看源码可以看见 WindowAssigner 这个抽象类有如下实现类：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-17-163413.png)

这些 WindowAssigner 实现类的作用介绍：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-16-155715.jpg)

如果你细看了上面图中某个类的具体实现的话，你会发现一个规律，比如我拿 TumblingEventTimeWindows 的源码来分析，如下：

```java
public class TumblingEventTimeWindows extends WindowAssigner<Object, TimeWindow> {
    //定义属性
    private final long size;
    private final long offset;

    //构造方法
    protected TumblingEventTimeWindows(long size, long offset) {
        if (Math.abs(offset) >= size) {
            throw new IllegalArgumentException("TumblingEventTimeWindows parameters must satisfy abs(offset) < size");
        }
        this.size = size;
        this.offset = offset;
    }

    //重写 WindowAssigner 抽象类中的抽象方法 assignWindows
    @Override
    public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
        //实现该 TumblingEventTimeWindows 中的具体逻辑
    }

    //其他方法，对外提供静态方法，供其他类调用
}
```

从上面你就会发现**套路**：

1、定义好实现类的属性

2、根据定义的属性添加构造方法

3、重写 WindowAssigner 中的 assignWindows 等方法

4、定义其他的方法供外部调用

### Window 组件之 Trigger 使用及源码分析

Trigger 表示触发器，**每个窗口都拥有一个 Trigger（触发器），该 Trigger 决定何时计算和清除窗口。**当先前注册的计时器超时时，将为插入窗口的每个元素调用触发器。在每个事件上，触发器都可以决定触发，即清除（删除窗口并丢弃其内容），或者启动并清除窗口。一个窗口可以被求值多次，并且在被清除之前一直存在。注意，在清除窗口之前，窗口将一直消耗内存。

说了这么一大段，我们还是来看看 Trigger 的源码，定义如下：

```java
public abstract class Trigger<T, W extends Window> implements Serializable {
    //当有数据进入到 Window 运算符就会触发该方法
    public abstract TriggerResult onElement(T element, long timestamp, W window, TriggerContext ctx) throws Exception;
    //当使用触发器上下文设置的处理时间计时器触发时调用
    public abstract TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception;
    //当使用触发器上下文设置的事件时间计时器触发时调用该方法
    public abstract TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception;
}
```

当有数据流入 Window 运算符时就会触发 onElement 方法、当处理时间和事件时间生效时会触发 onProcessingTime 和 onEventTime 方法。每个触发动作的返回结果用 TriggerResult 定义。继续来看下 TriggerResult 的源码定义：

```java
public enum TriggerResult {

    //不做任何操作
    CONTINUE(false, false),

    //处理并移除窗口中的数据
    FIRE_AND_PURGE(true, true),

    //处理窗口数据，窗口计算后不做清理
    FIRE(true, false),

    //清除窗口中的所有元素，并且在不计算窗口函数或不发出任何元素的情况下丢弃窗口
    PURGE(false, true);
}
```

查看源码可以看见 Trigger 这个抽象类有如下实现类：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-17-163751.png)

这些 Trigger 实现类的作用介绍：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-17-145735.jpg)

如果你细看了上面图中某个类的具体实现的话，你会发现一个规律，拿 CountTrigger 的源码来分析，如下：

```java
public class CountTrigger<W extends Window> extends Trigger<Object, W> {
    //定义属性
    private final long maxCount;

    private final ReducingStateDescriptor<Long> stateDesc = new ReducingStateDescriptor<>("count", new Sum(), LongSerializer.INSTANCE);
    //构造方法
    private CountTrigger(long maxCount) {
        this.maxCount = maxCount;
    }

    //重写抽象类 Trigger 中的抽象方法 
    @Override
    public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
        //实现 CountTrigger 中的具体逻辑
    }

    @Override
    public TriggerResult onEventTime(long time, W window, TriggerContext ctx) {
        return TriggerResult.CONTINUE;
    }

    @Override
    public TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception {
        return TriggerResult.CONTINUE;
    }
}
```

**套路**：

1. 定义好实现类的属性
2. 根据定义的属性添加构造方法
3. 重写 Trigger 中的 onElement、onEventTime、onProcessingTime 等方法
4. 定义其他的方法供外部调用

### Window 组件之 Evictor 使用及源码分析

Evictor 表示驱逐者，它可以遍历窗口元素列表，并可以决定从列表的开头删除首先进入窗口的一些元素，然后其余的元素被赋给一个计算函数，如果没有定义 Evictor，触发器直接将所有窗口元素交给计算函数。

我们来看看 Evictor 的源码定义如下：

```java
public interface Evictor<T, W extends Window> extends Serializable {
    //在窗口函数之前调用该方法选择性地清除元素
    void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
    //在窗口函数之后调用该方法选择性地清除元素
    void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
}
```

查看源码可以看见 Evictor 这个接口有如下实现类：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-17-163942.png)

这些 Evictor 实现类的作用介绍：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-17-153505.jpg)

如果你细看了上面三种中某个类的实现的话，你会发现一个规律，比如我就拿 CountEvictor 的源码来分析，如下：

```java
public class CountEvictor<W extends Window> implements Evictor<Object, W> {
    private static final long serialVersionUID = 1L;

    //定义属性
    private final long maxCount;
    private final boolean doEvictAfter;

    //构造方法
    private CountEvictor(long count, boolean doEvictAfter) {
        this.maxCount = count;
        this.doEvictAfter = doEvictAfter;
    }
    //构造方法
    private CountEvictor(long count) {
        this.maxCount = count;
        this.doEvictAfter = false;
    }

    //重写 Evictor 中的 evictBefore 方法
    @Override
    public void evictBefore(Iterable<TimestampedValue<Object>> elements, int size, W window, EvictorContext ctx) {
        if (!doEvictAfter) {
            //调用内部的关键实现方法 evict
            evict(elements, size, ctx);
        }
    }

    //重写 Evictor 中的 evictAfter 方法
    @Override
    public void evictAfter(Iterable<TimestampedValue<Object>> elements, int size, W window, EvictorContext ctx) {
        if (doEvictAfter) {
            //调用内部的关键实现方法 evict
            evict(elements, size, ctx);
        }
    }

    private void evict(Iterable<TimestampedValue<Object>> elements, int size, EvictorContext ctx) {
        //内部的关键实现方法
    }

    //其他的方法
}
```

发现**套路**：

1. 定义好实现类的属性
2. 根据定义的属性添加构造方法
3. 重写 Evictor 中的 evictBefore 和 evictAfter 方法
4. 定义关键的内部实现方法 evict，处理具体的逻辑
5. 定义其他的方法供外部调用

### 自定义 Window？

通过这几个源码，我们可以发现，它最后调用的都有一个方法，那就是 Window 方法，如下：

```java
//提供自定义 Window
public <W extends Window> WindowedStream<T, KEY, W> window(WindowAssigner<? super T, W> assigner) {
    return new WindowedStream<>(this, assigner);
}

//构造一个 WindowedStream 实例
public WindowedStream(KeyedStream<T, K> input,
        WindowAssigner<? super T, W> windowAssigner) {
    this.input = input;
    this.windowAssigner = windowAssigner;
    //获取一个默认的 Trigger
    this.trigger = windowAssigner.getDefaultTrigger(input.getExecutionEnvironment());
}
```

可以看到这个 Window 方法传入的参数是一个 WindowAssigner 对象（你可以利用 Flink 现有的 WindowAssigner，也可以根据上面的方法来自定义自己的 WindowAssigner），然后再通过构造一个 WindowedStream 实例（在构造实例的会传入 WindowAssigner 和获取默认的 Trigger）来创建一个 Window。

另外你可以看到滑动计数窗口，在调用 window 方法之后，还调用了 WindowedStream 的 evictor 和 trigger 方法，trigger 方法会覆盖掉你之前调用 Window 方法中默认的 trigger，如下：

```java
//滑动计数窗口
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
    return window(GlobalWindows.create()).evictor(CountEvictor.of(size)).trigger(CountTrigger.of(slide));
}

//trigger 方法
public WindowedStream<T, K, W> trigger(Trigger<? super T, ? super W> trigger) {
    if (windowAssigner instanceof MergingWindowAssigner && !trigger.canMerge()) {
        throw new UnsupportedOperationException("A merging window assigner cannot be used with a trigger that does not support merging.");
    }

    if (windowAssigner instanceof BaseAlignedWindowAssigner) {
        throw new UnsupportedOperationException("Cannot use a " + windowAssigner.getClass().getSimpleName() + " with a custom trigger.");
    }
    //覆盖之前的 trigger
    this.trigger = trigger;
    return this;
}
```

从上面的各种窗口实现，你就会发现了：Evictor 是可选的，但是 WindowAssigner 和 Trigger 是必须会有的，这种创建 Window 的方法充分利用了 KeyedStream 和 WindowedStream 的 API，再加上现有的 WindowAssigner、Trigger、Evictor，你就可以创建 Window 了，另外你还可以自定义这三个窗口组件的实现类来满足你公司项目的需求。



# 问题

##### 9、自定义窗口实现，不清楚。

```
assigner-元素分发器，trigger-窗口计算触发，evictor-元素剔除器(窗口计算动作前后，可自定义)
```

##### 10、常用算子对应不同的输入、输出流不清楚

| input           | output                     |              |
| --------------- | -------------------------- | ------------ |
| DataStream      | SingleOutputStreamOperator | map          |
| DataStream      | SingleOutputStreamOperator | flatMap      |
| DataStream      | SingleOutputStreamOperator | Filter       |
| DataStream      | KeyedDataStream            | KeyBy        |
| KeyedDataStream | SingleOutputStreamOperator | Reduce       |
| KeyedStream     |                            | Aggregations |
| KeyedStream     |                            | Window       |
|                 |                            | Union        |
|                 |                            | Window Join  |
|                 | SplitStream                | Split        |

##### 11、各种类型流的api，没有感官，需要实际使用

Split方法已经不推荐使用了！在 1.7 版本以后建议使用 Side Output 来实现分流操作。

##### 19、分流操作

filter、split都可做，但split不能多次分流

Side Output是如何进行连续分流？

通过以下方法，给元素打tag，产出SingleOutputStreamOperator

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- ProcessWindowFunction
- ProcessAllWindowFunction

##### 20、state两种类型和两种状态。。。看不懂蒙圈

##### 23、flink table 的模块功能