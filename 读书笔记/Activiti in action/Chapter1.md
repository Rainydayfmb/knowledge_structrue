# 第一章 认识activiti
## 1.1 什么是activitie
## 1.2 工作流基础
### 1.2.1什么是BPM
BPM是business process manager 缩写，中文含义是业务流程管理，是一套达成业务各种业环节整合的全面管理模式。

BPM包含两个层面的含义：**管理规范和软件工程**。

### 1.2.2 工作流的生命周期
- 定义：一般由业务需求人员进行，然后交给开发人员处理。
- 发布：由开发人员打包各种资源，然后在系统平台中发布流程定义。在具体的流程引擎中包含流程定义文件，自定义表单，任务监听类。
- 执行：按照流程的处理线路以任务驱动的方式执行业务流程。
- 监控：依赖执行阶段。业务人员在办理业务的同时收集每个任务的结果，然后根据结果作出相应的处理。
- 优化：流程优化。

### 1.2.3 什么是BPMN
business process model nation,业务流程建模标注。

## 1.3 activiti特点
**1.数据持久话**
actitiviti使用mybatis,从而可以通过sql执行commond。

**2.引擎Service接口**
activiti引擎支持七大service接口，可以通过processEngine获取，并且支持链式API的风格。

- RepositoryService:流程仓库service,用于管理流程仓库，例如部署，删除，读取流程资源。
- IdentifyService:身份service,可以管理和查询用户、组之间的关系。
RuntimeService:运行时service,可以处理所有正在运行状态的流程实例，任务等。
- TaskSerive:任务service，用于管理、查询任务，例如签收，办理，指派等。
- FormService:表单Service,用于读取和流程、任务相关的表单数据。
- HistoryService:历史Service,可以查询所有的历史数据，例如，流程实例、任务、活动、变量、附件等
- ManagerService:引擎可管理service,和具体的业务没有关系，主要可以查询引擎配置，数据库、作业等。

**3.流程设计器**
**4.原生支持Spring**
**5.分离运行时与历史数据**
在表结构设计方面也遵循运行时与历史数据的分离

## 1.4 activiti的应用
1.在系统集成方面的应用
- 与ESB企业服务总线整合，例如mule。
- 与规则引擎整合，例如JBoss Drools。
- 嵌入已有的系统平台。

## 1.5 activiti架构与组建
**Runntime:**
- activiti engine:作为最核心的模块，提供解析、执行、创建、管理（任务，流程示例）、查询历史记录并根据结果生成报表。

**Modeling:**
- activiti modeler:模型设计器，适用于业务人员把需求装换为规范流程定义。

- activiti designer:适用于开发模式


**Management:**
- activiti explorer:
可以用来管理仓库、用户、组，启动流程、任务办理等，此组建适用rest风格的api,提供一个基础的设计模型。

- activiti rest:
提供restful风格的服务，允许客户端以json的方式与引擎的rest api交互，通过的协议具有跨平台、跨语言的特性。



