<!-- TOC -->

- [1. 彻底了解大数据实时计算框架flink](#1-彻底了解大数据实时计算框架flink)
    - [1.1. 什么是flink?](#11-什么是flink)
    - [1.2. flink 整体架构](#12-flink-整体架构)
    - [1.3. flink支持多种部署方式](#13-flink支持多种部署方式)

<!-- /TOC -->
# 1. 彻底了解大数据实时计算框架flink
数据集类型
- 无穷数据集：无穷的持集成的数据集合；
- 有界数据集：有限不会改变的数据集合；

数据运算模型
- 流式：只要数据一直在生产，计算就持续的进行；
- 批处理：在预先定义的时间内运行计算，当计算完成时释放计算机资源；

## 1.1. 什么是flink?
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/pRMhfm.jpg)
flink是针对流数据和批数据的分布式处理引擎，是一款真正的流批统一的处理引擎。
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/vY6T3M.jpg)

## 1.2. flink 整体架构

![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/Drsi9h.jpg)

从下到上:
1.部署：Flink支持本地运行(IDE中直接运行程序)、能在独立集群(standalon模式)或者在yarn,mesos,k8s管理的集群上运行，也能部署在云上；
2.运行：Flink的核心是分布式流式数据引擎，意味着数据以一次一个时间的形式被处理。
3.API:DataStream,DataSet,Table,SQL API.
4.扩展库：Flink还包括用于CEPCEP(复杂时间处理)，机器学习、图形处理等场景。

## 1.3. flink支持多种部署方式
基于k8s的flink部署方式，运行架构有一下这种形式：
![avator](https://raw.githubusercontent.com/Aleksandr-Filichkin/flink-k8s/master/flow.jpg)

flink分布式运行
flink作业提交架构流程图如下图：
![avator](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/p92UrK.jpg)

1.program code:我们编写的flink应用程序代码。

2.job client:他不是flink执行的内部部分，但它是任务执行的起点。Job Client负责接受用户的程序代码，然后创建数据流，将数据交给jobMnager以便进一步的处理。执行完成后，jobclient将结果返回给用户。

3.Job manager:主进程（也称作作业管理器）协调和管理程序的执行。它的主要指责包括任务安排、管理checkpoint、故障恢复等。机器集群中至少要有一个master，master负责调度task、协调checkpoints和容灾，高可用的话可以有多个mastermaster，其他的是standby。job manager包含actor system、scheduler、check pointing三个重要组件。

4.Task Manager:从job manager处接收需要部署的Task。task manager是在jvm中一个或多个线程中执行任务的节点工作。任务执行的并行性由每个Task Mnager的可用的任务槽决定。每个任务代表分配给任务槽的一组资源。例如，如果 Task Manager 有四个插槽，那么它将为每个插槽分配 25％ 的内存。可以在任务槽中运行一个或多个线程。同一插槽中的线程共享相同的 JVM。 


