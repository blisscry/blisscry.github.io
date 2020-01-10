---

layout:     post
title:      从paxos到zookeeper
subtitle:   分布式
date:       2020-01-09
author:     skaleto
catalog: true
tags:
    - 分布式

---



## Paxos

paxos算法的基本流程如下图，蓝色为prepare阶段，黄色为accept阶段

![paxos](..\img\paxos&zookeeper\paxos.png)





## Zab

Zab借鉴了paxos，是专门为zookeeper设计的支持崩溃恢复的原子广播协议

### 核心

所有事务请求必须经由全局唯一的服务器来协调处理，即Leader服务器，其余的服务器成为Follower服务器

### 消息广播

是一个简化的2PC过程

![ZAB-消息广播](..\img\paxos&zookeeper\ZAB-消息广播.png)



### 崩溃恢复

#### 已处理的消息不能丢

当集群中的leader崩溃时，集群进入崩溃恢复模式，其他follower中选出一个具有最大ZXID的机器（ZXID越大，意味着接收到的proposal最新，越可能包含完整的信息），此时这台机器升级为新的leader，其他机器成为新的follower。

新的leader与follower建立通信，将follower中没有的proposal发布过去，并紧跟一个COMMIT，因为这些proposal理应已经被处理了，最后所有设备都保持和leader一致的状态。



#### 被丢弃的消息不能再次出现

当新的leader重启再次假如集群时，他自身可能带有一些还没有来得及发出的proposal，那么他的ZXID肯定大于恢复时的ZXID，但其实他们是不需要的proposal，如何解决呢？

Zab 通过巧妙的设计 ZXID来实现这一目的。一个 zxid 是64位，高 32 是纪元（epoch）编号，每经过一次 leader 选举产生一个新的 leader，新 leader 会将 epoch 号 +1。低 32 位是消息计数器，每接收到一条消息这个值 +1，新 leader 选举后这个值重置为 0。这样设计的好处是旧的 leader 挂了后重启，它不会被选举为 leader，因为此时它的 zxid 肯定小于当前的新 leader。当旧的 leader 作为 follower 接入新的 leader 后，新的 leader 会让它将所有的拥有旧的 epoch 号的未被 COMMIT 的 proposal 清除。   





