# Chapter24 容错重启策略

## 重启策略restartStrategies：

默认noReastart

固定重启次数策略 fixDealyRestart()

频率重启次数策略 failureRateRestart()

开启checkpoint行为enableCheckPointing()，开启后，作业是fixedDealy策略，但如果程序只用到了无状态算子，例如map，那checkpoint只起到了重启策略的作用

## 恢复策略failureoverStartegyStartegy

目前flink，流作业都是整个作业全部重启，批作业支持局部重启

