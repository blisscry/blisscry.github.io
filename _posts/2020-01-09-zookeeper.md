---

layout:     post
title:      ZooKeeper
subtitle:   ZooKeeper
date:       2019-12-26
author:     skaleto
catalog: true
tags:
    - java，jni

---



# ZooKeeper



## Zookeeper的应用场景

- 配置管理
- DNS服务
- 组成员管理
- 分布式锁



ZooKeeper的数据模型（Google chubby也是这么做的），

层次模型常见于文件系统；key-value常见于数据库



znode的类型：

- 持久性znode
- 临时性znode
- 持久顺序性znode
- 临时顺序性znode



Master-worker架构

广泛使用的分布式架构，系统中有一个master负责监控worker的 状态，并为worker分配任务

- 在任意时刻，系统中不能出现两个master，多个master会脑裂
- 有active master和backup master，如果active失败，则backup快速进入active
- master监控worker的状态，有新成员加入时，进行通知和工作分配



使用master-worker的组件

- HBase（HMaster，HRegionServer）
- kafka（Controller，broker）
- HDFS（NameNode，DataNode）



如何使用Zookeeper实现master-worker？

使用watch机制



zookeeper集群有两种模式

- standalone，单点，一般用于开发
- quorum，多节点



Session

客户端会和集群中的某个节点创建session，当客户端关闭或长时间没有消息时，session会关闭，可以重连



Quorum模式

多节点，leader节点可以处理读写请求，follower只可以处理读请求，follower在接收到写请求时会把消息转发给leader来处理



数据一致性

- 先到达leader的写请求会被先处理，leader决定写请求的执行顺序
- 客户端FIFO顺序，来自给定客户端的请求按照发送顺序执行



watch机制

![image-20200110002918988](/Users/yaoyibin/Library/Application Support/typora-user-images/image-20200110002918988.png)



实现一个分布式队列

1. 使用顺序持久节点来存放队列中的元素
2. 入队方法：在队列节点下创建一个新的持久顺序节点（由于持久顺序节点的特性，id会自动增加）
3. 出队方法：遍历队列节点下的子节点



实现一个分布式锁