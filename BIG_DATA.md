# 大数据

## 流式计算框架
[参考](https://bigdata.163yun.com/product/article/5)
- flink
- Spark
- strom
### flink架构及特性分析
flink是针对流数据和批数据分布式处理的引擎，在某些对实时性要求非常高的场景，基本上都是采用Flink来作为计算引擎，它不仅可以处理有界的批数据，还可以处理无界的流数据，**在Flink的设计意愿就是将批处理当做成是流处理的一种特例。**
#### 1.1 基本架构
flink的基本架构与spark类似，是一个基于Master-slave风格的架构。
![avatar](http://nos.netease.com/knowledge/ba8c8e77-7516-4df5-b4d5-5969209b7dca)
当flink应用启动后，首先会启动一个JobManager和一个或多个的TaskManager。由client提交任务给JobManager,JobManager在调度任务到各个TaskManager去执行，然后将心跳和统计信息汇报给 JobManager。TaskManager之间以流的形式进行数据的传输。上述均为独立的JVM进程。




### spark
Apache spark 是一种包含流处理能力的下一代批处理框架。与hadoop的mapreduce引擎基于各种相同原则开发而来的spark主要侧重于通过完善的内存计算和处理优化机制加快批处理工作负载的运行速度。

Spark可以作为独立集群部署（需要响应存储层的配合），或可与hadoop继承并取代mapreduce引擎；

### spark streaming
![avatar](/Users/feipeng/Desktop/图片/WechatIMG1.jpeg)
spark streaming 是spark api 核心的扩展，可实现实时数据的快速扩展，高吞吐量，容错处理。数据可以从很多来源（如kafka,Flume,kinesis等）中提取，并且可以通过很多函数来处理这些数据，处理完后的数据可以直接存入数据库或者Dashboard等；
![avatar](https://user-gold-cdn.xitu.io/2018/5/16/16367d3042c62eac?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

sparkStreaming 的内部实现原理是接收实时输入数据流并将数据分成批处理，然后由spark引擎处理以批量生成最终结果流，也就是常说的micro-batch模式。

### spark streaming的优缺点
**优点**：
- spark streaming 内部的实现和调度方式高度依赖spark的DAG调度器和RDD，这就决定了spark streaming 的设计初衷必须是粗粒度方式的，也就是无法做到真正的实时处理。
- spark steaming 的粗力度执行方式使其确保处理且近处理一次的特性，同时也可以更方便地实现容错恢复机制。
- 由于spark streaming 的 Dstream的本质是RDD在流式数据上的抽象，因此基于RDD的各种操作也有相应的基于Dstream的版本，这样就大大降低了用户对新框架的学习成本，在了解spark的情况下用户将很容易使用spark streaming 。

**缺点**
- spark streaming 的粗力度处理方式也造成了不可避免的数据延迟。在粗力度处理方式下，理想情况下每一条记录都会被实时处理，而在spark streaming 中，数据需要汇总到一定的量后在一次性处理，这就增加数据处理的延迟，这种延迟是由框架的设计引入的，并不是由网络或其他情况造成的。
- 使用的是processing time 而不是event time.

**三种时间模式**
![avatar](https://upload-images.jianshu.io/upload_images/5501600-16fb5013ecde729f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
从这张图里可以很清楚的看到三种Time模型的区别。
- EventTime是数据被生产出来的时间，可以是比如传感器发出信号的时间等（此时数据还没有被传输给flink）。
- IngestionTime是数据进入flink的时间，也就是从Source进入flink流的时间（此时数据刚刚被传给flink）
- ProcessingTime是针对当前算子的系统时间，是指该数据已经进入某个operator时，operator所在系统的当前时间


#### 介绍
首先，什么是流（streaming）？数据流是连续到达的无穷序列。流处理将不断流动的输入数据分成独立的单元进行处理。流处理是对流数据的低延迟处理和分析。Spark Streaming是Spark API核心的扩展，可实现实时数据的快速扩展，高吞吐量，高容错处理。Spark Streaming适用于大量数据的快速处理。实时处理用例包括：网站监控，网络监控欺诈识别网页点击广告物联网传感器
- 网站监控，网络监控
- 欺诈识别
- 网页点击
- 广告
- 物联网传感器

Spark Streaming是Spark生态系统当中一个重要的框架，它建立在Spark Core之上，下面这幅图也可以看出Sparking Streaming在Spark生态系统中地位。
![avatar](https://img-blog.csdn.net/20180512122140705?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3MTQyMzQ2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


Spark Streaming支持如HDFS目录，TCP套接字，Kafka，Flume，Twitter等数据源。数据流可以用Spark 的核心API，DataFrames SQL，或机器学习的API进行处理，并且可以被保存到HDFS，databases或Hadoop OutputFormat提供的任何文件系统中去。
![avatar](https://user-gold-cdn.xitu.io/2018/5/16/16367d3042c62eac?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**spark streaming如何工作**
Spark Streaming将数据流每X秒分作一个集合，称为Dstreams，它在内部是一系列RDD。您的Spark应用程序使用Spark API处理RDD，并且批量返回RDD操作的结果。
![avatar](https://user-gold-cdn.xitu.io/2018/5/16/16367d3042c9187d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**示例应用程序的体系结构**
![avatar](https://user-gold-cdn.xitu.io/2018/5/16/16367d3042e9566a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



#### 2.1基本架构
![avatar](http://nos.netease.com/knowledge/4b9dcd06-973f-4e01-a596-a9e90af3f6e4)
<center>基于是spark core的spark streaming架构</center>
sparkStreaming 是将流式计算分解成一些列短小的批处理作业。这里的批处理引擎是spark,也就是把spark streaming的输入数据按照batch size(如一秒)分成一段一段的数据（Discretized Stream),每一段数据都转换成Spark中的RDD（Resilient Distributed Dataset），然后将Sparking Streaming中对DStream的Transformation操作变为针对Spark中对RDD的Transformation操作，将RDD经 过操作变成中间结果保存在内存中。整个流式计算根据业务的需求可以对中间的结果进行叠加，或者存储在外部设备。

![avatar](http://nos.netease.com/knowledge/bfd5f111-b93e-41f4-a0c5-ec6ce63ec456)
简而言之，Spark Streaming把实时输入数据流以时间片Δt（如1秒）为单位切分成块，Spark Streaming会把每块数据作为一个RDD，并使用RDD操作处理每一小块数据。每个块都会生成一个Spark Job处理，然后分批次提交job到集群中去运行，运行每个job的过程和真正的spark 任务没有任何区别。
![avatar](https://img-blog.csdn.net/2018072718501344?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FjY3B0YW5nZ2FuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**JobScheduler**
负责job的调度，JobScheduler是SparkStreaming 所有Job调度的中心， JobScheduler的启动会导致ReceiverTracker和JobGenerator的启动。其中
（1）ReceiverTracker的启动导致运行在Executor端的Receiver启动并且接收数据，ReceiverTracker会记录Receiver接收到的数据meta信息。
（2）JobGenerator的启动导致每隔BatchDuration，就调用DStreamGraph生成RDD Graph，并生成Job。  JobScheduler中的线程池来提交封装的JobSet对象。Job中封装了业务逻辑，导致最后一个RDD的action 被触发，被DAGSchedule真正调度在spark集群上执行该job。

**JobGenerator**
负责Job的生成
通过定时器每隔一段时间根据Dstream的依赖关系生一个一个DAG图。


**ReceiverTracker**
负责数据的接收，管理和分配
ReceiverTracker在启动Receiver的时候他有ReceiverSupervisor,其实现是ReceiverSupervisorImpl, ReceiverSupervisor本身启动的时候会启动Receiver，Receiver不断的接收数据，通过BlockGenerator将数据转换成Block。定时器会不断的把Block数据通过BlockManager或者WAL进行存储，数据存储之后ReceiverSupervisorImpl会把存储后的数据的元数据Metadate汇报给ReceiverTracker,其实是汇报给ReceiverTracker中的RPC实体ReceiverTrackerEndpoint，主要。
