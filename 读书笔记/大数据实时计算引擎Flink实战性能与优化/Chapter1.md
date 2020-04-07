<!-- TOC -->

- [1. 公司到底需不需要引入实时计算引擎](#1-公司到底需不需要引入实时计算引擎)
    - [1.1. 实时计算需求](#11-实时计算需求)
    - [1.2. 实时计算场景](#12-实时计算场景)

<!-- /TOC -->
# 1. 公司到底需不需要引入实时计算引擎
## 1.1. 实时计算需求
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/h8Jmtt.jpg)
最根本的需求都是要实时查看数据信息，要考虑的问题分为三点
- 如何实时采集数据
- 如何实时计算
- 如何实时下发

![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/NQzEtY.jpg)

## 1.2. 实时计算场景
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/zL93nD.jpg)

**1.实时数据存储**
做一些微聚合、过滤某些字段、数据脱敏，组建数据仓库，实时ETL.

**2.实时数据分析**
实时数据接入tensorflow等机器学习框架或者一些算法进行数据建模、分析，然后动态的给出商品推荐、广告推荐。

**3.实时监控告警**
金融相关涉及交易、实时风控、车流量预警、服务器监控告警、应用日志告警

**4.实时数据报表**
活动营销实时营销售大屏，TopN商品

**离线计算VS实时计算**

流处理与批处理

![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/VN7lQm.jpg)

实时计算场景
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/SrubtS.jpg)

实时数据需要不断的从MQ中读取采集的数据，然后处理计算后往DB里面存储，在计算这层你无法感知到会有多少数据量过来，要做一些简单的操作(过滤、聚合等等)、即使数据下发

离线计算场景
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/eseUjV.jpg)

在计算这层，他从DB里面读取数据，该数据一般就是固定的（前一天，前一个月，前一个星期），在做一些复杂的计算或者统计分析，最后生成可供观察的报表

离线计算和实时计算的特点 
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/g4OSIs.jpg)

