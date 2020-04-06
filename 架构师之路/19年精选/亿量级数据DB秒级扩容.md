<!-- TOC -->

- [1. 亿级数据DB秒级平滑扩容](#1-亿级数据db秒级平滑扩容)
    - [1.1. 如何实现数据库的高可用](#11-如何实现数据库的高可用)
    - [1.2. 如何应对数据量的暴增？](#12-如何应对数据量的暴增)
    - [1.3. 高可用的互联网微服务分层的架构](#13-高可用的互联网微服务分层的架构)
    - [1.4. 数据量持续变大，解决数据库压力过大问题](#14-数据量持续变大解决数据库压力过大问题)
        - [1.4.1. 要增加到2*n个库，数据库如何平滑迁移扩容，并保证对外的可用性](#141-要增加到2n个库数据库如何平滑迁移扩容并保证对外的可用性)
            - [1.4.1.1. 方案一：停服扩容](#1411-方案一停服扩容)
            - [1.4.1.2. 停服扩容回滚步骤](#1412-停服扩容回滚步骤)
            - [1.4.1.3. 停服方案优势](#1413-停服方案优势)
            - [1.4.1.4. 停服方案缺点](#1414-停服方案缺点)
            - [1.4.1.5. 方案二：平滑扩容](#1415-方案二平滑扩容)
                - [1.4.1.5.1. 步骤一：修改配置](#14151-步骤一修改配置)
                - [1.4.1.5.2. 步骤二： reload配置，实例扩容。](#14152-步骤二-reload配置实例扩容)
                - [1.4.1.5.3. 步骤三： 收尾工作，数据收缩](#14153-步骤三-收尾工作数据收缩)
            - [1.4.1.6. 平滑扩容总结](#1416-平滑扩容总结)

<!-- /TOC -->
# 1. 亿级数据DB秒级平滑扩容
数据库上层都有一个微服务，服务层记录“业务库”与“数据库实例配置”的映射关系，通过数据库连接池向数据库路由sql语句。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTDicxhdHS3wrArZfuzkc7OUOLAu8CzY2SQU6lFwNXOicibC3sSAqFSr4pA/640?wx_fmt=png)

如上图所示，服务层配置用户库user对应的数据库实例ip。其实就是一个内网的域名。

## 1.1. 如何实现数据库的高可用
数据库高可用，很常见的一种方式，使用双主同步+keepalived+虚ip的方式进行。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTWicwsXpucQ1QMZ1WLkyyKEzJzNtRRHLHr5Lse93ScRHMics7rKDJnVhg/640?wx_fmt=png)

如上图所示，两个相互同步的主库使用相同的虚ip。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTDpNcQjHJ6lhzR3vbmRnhx32QEISagDblQibfP8ER7UEdWBG2vkrysmA/640?wx_fmt=png)
当主库挂掉的时候，虚ip自动漂移到另一个主库，整个过程对调用方透明，通过这种方式保证数据库的高可用。

## 1.2. 如何应对数据量的暴增？
随着数据量的增大，数据库要进行水平切分，分库后将数据分布到不同的数据库实例（甚至物理机器）上，以达到降低数据量，增强性能的扩容目的。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGT2zfBUuaB18yMn41ETMv9YhLcGCDyEEOib3s00yVnWuTSib1uP0CgwaKg/640?wx_fmt=png)
如上图所示，用户库user分布在两个实例上，ip0和ip1，服务层通过用户标识uid取模的方式进行寻库路由，模2余0的访问ip0上的user库，模2余1的访问ip1上的user库。**水平切分集群的读写实例加倍，单个实例的数据量减半，性能增长可不止一倍。**

## 1.3. 高可用的互联网微服务分层的架构
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTdclkCtxvlGCicTicjaA4MRhcYhsrb9zRNycIWMJpNn1gSSq1PMPicsArQ/640?wx_fmt=png)
又有水平切分，又保证高可用

## 1.4. 数据量持续变大，解决数据库压力过大问题
此时，需要继续水平拆分，拆成更多的库，降低单库数据量，增加库主库实例（机器）数量，提高性能。
### 1.4.1. 要增加到2*n个库，数据库如何平滑迁移扩容，并保证对外的可用性
#### 1.4.1.1. 方案一：停服扩容
**停服扩容的步骤：**
1. 停服公告
2. 停止服务，数据不再写入
3. 新建2*n个新库，并做好高可用
4. 写一个小脚本进行数据迁移，把数据从n个库里面select出来，insert到2* n个库里面；
5. 修改为服务的数据库路由配置，模n变为2n；
6. 微服务重启，连接新库重新对外开发服务

整个过程，最耗时的是第四步数据迁移

#### 1.4.1.2. 停服扩容回滚步骤
如果数据迁移失败，或者迁移后测试失败，则将配置改回旧库，恢复服务即可。

#### 1.4.1.3. 停服方案优势
简单

#### 1.4.1.4. 停服方案缺点
（1）需要停止服务，方案不高可用；
（2）技术同学压力大，所有工作要在规定时间内完成，根据经验，压力越大约容易出错；
（3）如果有问题第一时间没检查出来，启动了服务，运行一段时间后再发现有问题，则难以回滚，如果回档会丢失一部分数据；

#### 1.4.1.5. 方案二：平滑扩容
平滑扩容方案
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTdclkCtxvlGCicTicjaA4MRhcYhsrb9zRNycIWMJpNn1gSSq1PMPicsArQ/640?wx_fmt=png)
**平滑扩容步骤：**
##### 1.4.1.5.1. 步骤一：修改配置
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTtQJjJa3MmwzsXwUNBwIdEDEIpa4F4p4zxWT9qIv1yNRWP265kMx6rw/640?wx_fmt=png)
主要修改两处：
1.数据库实例所在的机器做**双虚ip**
（1）原%2=0的库是虚ip0，现增加一个虚ip00；
（2）原%2=1的库是虚ip1，现增加一个虚ip11；

2.修改服务的配置，将2个库的数据库配置，改为4个库的数据库配置，修改的时候要注意旧库与新库的映射关系：
（1）%2=0的库，会变为%4=0与%4=2；
（2）%2=1的部分，会变为%4=1与%4=3；

##### 1.4.1.5.2. 步骤二： reload配置，实例扩容。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTYdtyLG7P9D4XsqP2EXJ2FaV0Qo4KXTicjDicrZs70B6TEdKA0ic6fNLpA/640?wx_fmt=png)
服务层reload配置，reload可能是这么几种方式：
（a）比较原始的，重启服务，读新的配置文件；
（b）高级一点的，配置中心给服务发信号，重读配置文件，重新初始化数据库连接池；

不管哪种方式，reload之后，数据库的实例扩容就完成了，原来是2个数据库实例提供服务，现在变为4个数据库实例提供服务，这个过程一般可以在秒级完成。
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTYdtyLG7P9D4XsqP2EXJ2FaV0Qo4KXTicjDicrZs70B6TEdKA0ic6fNLpA/640?wx_fmt=png)
整个过程可以逐步重启，对服务的正确性和可用性完全没有影响：
（a）即使%2寻库和%4寻库同时存在，也不影响数据的正确性，因为此时仍然是双主数据同步的；
（b）即使%4=0与%4=2的寻库落到同一个数据库实例上，也不影响数据的正确性，因为此时仍然是双主数据同步的；
完成了实例的扩展，会发现每个数据库的数据量依然没有下降，所以第三个步骤还要做一些收尾工作。

##### 1.4.1.5.3. 步骤三： 收尾工作，数据收缩
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTIahl7W9Tf8Jib5aeIWExUDyvgXXcDaNOpj4YmHoLlqVethQB5F7U6ibg/640?wx_fmt=png)
（a）把双虚ip修改回单虚ip；
（b）解除旧的双主同步，让成对库的数据不再同步增加；
（c）增加新的双主同步，保证高可用；
（d）删除掉冗余数据，例如：ip0里%4=2的数据全部删除，只为%4=0的数据提供服务；

#### 1.4.1.6. 平滑扩容总结
![avator](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/YrezxckhYOzOGwoVobJtZTvibPmgp6DGTmBTw0nicWUrg8mrrkNHqHattRQgNYjwV3dj52yQ3RJ36wWhot4gZ8vg/640?wx_fmt=png)

互联网大数据量，高吞吐量，高可用微服务分层架构，数据库实现秒级平滑扩容的三个步骤为：
（1）修改配置（双虚ip，微服务数据库路由）；
（2）reload配置，实例增倍完成；
（3）删除冗余数据等收尾工作，数据量减半完成；

 
