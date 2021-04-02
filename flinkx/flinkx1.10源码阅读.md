

# Java方法用途

Class

```java
Class.getDeclaredFields //获取类属性
Class.getFields //获取类属性
/**
getFields()：获得某个类的所有的公共（public）的字段，包括父类中的字段。 
getDeclaredFields()：获得某个类的所有声明的字段，即包括public、private和proteced，但是不包括父类的申明字段。

同样类似的还有getConstructors()和getDeclaredConstructors()、getMethods()和getDeclaredMethods()，这两者分别表示获取某个类的方法、构造函数。
**/
```



# flinkx启动流程-Launcher

1.Launcher类main方法，new OptionParse对象，构造方法中调initOptions-》addOptions(args)，传入命令参数



##### 【com/dtstack/flinkx/options/OptionParser.java:58】

Options类的属性使用了接口OptionRequired，去构造任务参数Options对象时，通过getDeclaredFields反射获取java.lang.reflect.Field，再通过getAnnotation获取注释，再调用org.apache.commons.cli.Options#addOption(java.lang.String, boolean, java.lang.String)构造Options

org.apache.commons.cli.Parser#parse(org.apache.commons.cli.Options, java.lang.String[]) 构造cmd并返回(此时shell命令参数已构建到cmd)，将cmd传入com.dtstack.flinkx.options.OptionParser#initOptions，遍历cmd的参数，刷新properties

```java
private CommandLine addOptions(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, ParseException {
    //properties对象是由new com.dtstack.flinkx.options.Options构造方法创建，赋予默认属性值
  	Class cla = properties.getClass(); 
    Field[] fields = cla.getDeclaredFields();
  //将properties对象属性值加载到org.apache.commons.cli.Options类对象options中
    for(Field field:fields){
        String name = field.getName();
        OptionRequired optionRequired = field.getAnnotation(OptionRequired.class);
        if(optionRequired != null){
            options.addOption(name,optionRequired.hasArg(),optionRequired.description());
        }
    }
  //parser对象为new BasicParser()构造，BasicParser extends Parser。将options对象和命令参数args传入解析成CommandLine对象
    CommandLine cl = parser.parse(options, args);
    return cl;
}
```

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface OptionRequired {

	boolean required() default false;

	boolean hasArg() default true;

	String description() default "";
}
```

##### 【com.dtstack.flinkx.options.OptionParser#getProgramExeArgList】

读取properties配置信息，拿到json文件存储路径，com.dtstack.flinkx.options.OptionParser#getProgramExeArgList方法中，同时获取job的json配置等，返回数组

##### 【com/dtstack/flinkx/launcher/Launcher.java:79】

将任务配置数组转hashMap，查找-p参数，并进行替换（自定义入参，用于替换脚本中的占位符，如脚本中存在占位符${pt1},\${pt2}，则-p参数可配置为pt1=20200101,pt2=20200102）

##### 选择flinkx启动模式【com/dtstack/flinkx/launcher/Launcher.java:90】

switch case启动模式，根据com.dtstack.flinkx.options.Options#getMode的值来启动

##### 【com.dtstack.flinkx.Main.main】

调用com.dtstack.flinkx.Main.main

```java
switch (ClusterMode.getByName(launcherOptions.getMode())) {
    case local:
        com.dtstack.flinkx.Main.main(argList.toArray(new String[0]));
        break;
    case standalone:
```

##### 【com.dtstack.flinkx.Main#main】

```java
public class Main {

    public static Logger LOG = LoggerFactory.getLogger(Main.class);

    public static final String READER = "reader";
    public static final String WRITER = "writer";
    public static final String STREAM_READER = "streamreader";
    public static final String STREAM_WRITER = "streamwriter";

    private static final ObjectMapper objectMapper = new ObjectMapper();

    public static void main(String[] args) throws Exception {
        com.dtstack.flinkx.options.Options options = new OptionParser(args).getOptions();
        String job = options.getJob();
        String jobIdString = options.getJobid();
        String monitor = options.getMonitor();
        String pluginRoot = options.getPluginRoot();
        String savepointPath = options.getS();
        String remotePluginPath = options.getRemotePluginPath();
        Properties confProperties = parseConf(options.getConfProp());

        // 解析jobPath指定的任务配置文件
        DataTransferConfig config = DataTransferConfig.parse(job);
        speedTest(config);

        if(StringUtils.isNotEmpty(monitor)) {
            config.setMonitorUrls(monitor);
        }

        if(StringUtils.isNotEmpty(pluginRoot)) {
            config.setPluginRoot(pluginRoot);
        }

        if (StringUtils.isNotEmpty(remotePluginPath)) {
            config.setRemotePluginPath(remotePluginPath);
        }
				//读取flink-conf.yaml，加载flink配置
        Configuration flinkConf = new Configuration();
        if (StringUtils.isNotEmpty(options.getFlinkconf())) {
            flinkConf = GlobalConfiguration.loadConfiguration(options.getFlinkconf());
        }
				
        StreamExecutionEnvironment env = (StringUtils.isNotBlank(monitor)) ?
                StreamExecutionEnvironment.getExecutionEnvironment() :
                new MyLocalStreamEnvironment(flinkConf);

        env = openCheckpointConf(env, confProperties);
        configRestartStrategy(env, config);

        SpeedConfig speedConfig = config.getJob().getSetting().getSpeed();
				//flink env加载flinkx插件地址列表到classpath
        PluginUtil.registerPluginUrlToCachedFile(config, env);

        env.setParallelism(speedConfig.getChannel());
        env.setRestartStrategy(RestartStrategies.noRestart());
       	//flink环境配置完成，详细调用内容看在下面的【com.dtstack.flinkx.reader.DataReaderFactory#getDataReader】
        BaseDataReader dataReader = DataReaderFactory.getDataReader(config, env);
      	//MysqlReader继承JdbcDataReader，实际调用JdbcDataReader#readData方法
        DataStream<Row> dataStream = dataReader.readData();
        if(speedConfig.getReaderChannel() > 0){
            dataStream = ((DataStreamSource<Row>) dataStream).setParallelism(speedConfig.getReaderChannel());
        }

        if (speedConfig.isRebalance()) {
            dataStream = dataStream.rebalance();
        }

        BaseDataWriter dataWriter = DataWriterFactory.getDataWriter(config);
        DataStreamSink<?> dataStreamSink = dataWriter.writeData(dataStream);
        if(speedConfig.getWriterChannel() > 0){
            dataStreamSink.setParallelism(speedConfig.getWriterChannel());
        }

        if(env instanceof MyLocalStreamEnvironment) {
            if(StringUtils.isNotEmpty(savepointPath)){
                ((MyLocalStreamEnvironment) env).setSettings(SavepointRestoreSettings.forPath(savepointPath));
            }
        }

        JobExecutionResult result = env.execute(jobIdString);
        if(env instanceof MyLocalStreamEnvironment){
            ResultPrintUtil.printResult(result);
        }
    }
}
```

##### 【com.dtstack.flinkx.reader.DataReaderFactory#getDataReader】

```java
public class DataReaderFactory {

    private DataReaderFactory() {
    }

    public static BaseDataReader getDataReader(DataTransferConfig config, StreamExecutionEnvironment env) {
        try {
            String pluginName = config.getJob().getContent().get(0).getReader().getName();
          	//拼接Reader插件类名，例如com.dtstack.flinkx.mysql.reader.MysqlReader
            String pluginClassName = PluginUtil.getPluginClassName(pluginName);
            Set<URL> urlList = PluginUtil.getJarFileDirPath(pluginName, config.getPluginRoot(), null);
						//如何反射的？
            return ClassLoaderManager.newInstance(urlList, cl -> {
                Class<?> clazz = cl.loadClass(pluginClassName);
              //获取构造方法的构造器
                Constructor constructor = clazz.getConstructor(DataTransferConfig.class, StreamExecutionEnvironment.class);
              //传入构造方法参数，创建对应的Reader对象，这里是MysqlReader,实际上调用com.dtstack.flinkx.mysql.reader.MysqlReader#MysqlReader的构造方法，同时MysqlReader继承JdbcDataReader，
                return (BaseDataReader)constructor.newInstance(config, env);
            });
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

# 问题

- *Annotation自定义元素属性描述 注释接口为什么需要 @Retention(RetentionPolicy.RUNTIME)*
- OptionParser类中，options、parser、properties是干嘛的
- com.dtstack.flinkx.reader.DataReaderFactory#getDataReader中如何反射Reader