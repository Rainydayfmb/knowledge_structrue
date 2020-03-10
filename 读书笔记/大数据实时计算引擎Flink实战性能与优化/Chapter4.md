# flink-wordcount demo
## WordCount应用程序代码分析
1.创建好StreamExecutionEnviroment(流程序的运行环境)
```java
StreamExcutionEnviroment env = StreamExcutionEnviromentnviroment.getExcutionEnviroment();
```

2.给流程序的运行环境设置全局变量(args中获取)
```
env.getConfig().setGlobalJobParameters(ParamenterTool.fromArgs(args));
```

3.构建数据源
```
env.fromElements(WORDS);
```

4.将字符串进行分隔然后收集

