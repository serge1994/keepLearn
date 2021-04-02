Flinkx 学习-编译使用入门

配置mvn环境变量

cd flinkx/bin  执行jar.sh 安装依赖jar

cd flinkx目录，mvn clean package -Dmaven.test.skip=true 进行编译

./bin/flinkx -mode local -job stream2stream.json -pluginRoot ../plugins/

- 

