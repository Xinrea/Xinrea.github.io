---
layout: post
title: "Google File System 学习01"
date: 2018-07-27
tags: google filesystem distributed-system
---
## 1 介绍

* 节点失效被看成是正常情况，而不再视为异常情况
* 文件都是非常巨大的
* 大部分文件都只会在文件尾新增数据
* 与应用一起设计文件系统API

## 2 设计概览

### 2.1 约定

* 系统是建立在大量廉价的计算机上，故障经常发生在这些计算机上
* 系统储存了许多超大文件，系统对超大文件管理进行特殊优化，支持小文件的管理
* 一般需要应对两类读取：大的流式读取和小规模的随机读取，随机读取可以在应用里进行优化，将对需读取的内容进行排序，以确保单向顺序读取，提高读取效率
* 明确对多客户端并行操作同一个文件的支持
* 高性能的稳定带宽比低延迟更加重要

### 2.2 接口

* 常见的文件操作：create, delete, open, close, read, write等
* snapshot, record append等

### 2.3 架构

```
master ---- chunkserver01
     \----- chunkserver02
      \---- chunkserver03
       ...
```

* 集群由一个master和多个chunkserver组成
* 集群会被许多client同时访问
* 节点都只是普通的linux PC
* 文件都被拆分为固定大小的chunk
* master根据chunk产生的时间，生成永久不变的64位chunk handle标志
* chunkserver上简单的使用LinuxFileSystem来管理块
* 块在不同的chunkserver上有备份
* master只保留元数据
* client只缓存元数据
* master与chunkserver使用心跳线通讯，传递信息以及查询状态
* client在master获取元数据后，后续数据的通讯直接与chunkserver进行

### 2.4 单个master

* 尽量减少master的读写，避免成为瓶颈

```
client: offset->trunk index (chunk size固定)
client--------filename,index--------->master (可以设置获得多个chunk的信息)
client<-------handle,location---------master (返回的chunk信息会被cache)
client--------handle,byte range------>chunkserver (使用location来与其建立连接)
client<-------data--------------------chunkserver
```

### 2.5 chunk块大小

* 设置为64M
* 减少了元数据大小，可直接保存在master的内存中
* 焦点问题：小文件只有1个chunk，多客户端同时访问时，保存这个chunk的chunkserver们成为焦点；对于大文件则没有这个问题，大文件拥有非常多的chunk，每个chunk保存的位置各不相同。在batch-queue(job-queue)下使用GFS时，worker启动时加载可执行程序会导致焦点问题，可以通过增加程序副本数量、错开worker启动时间来解决；永久解决方案：客户端能够互相读取数据。

### 2.6 元数据

主要包含以下三种类型的数据：
* file和chunk的namespace
* 文件到chunk的映射关系
* chunk副本的位置

前两种类型的数据变化记录在master磁盘上的operation log中，log会在远端机器上保留副本。第三种类型的数据在master启动或者chunkserver加入集群时，向chunkserver询问获得。

#### 2.6.1 内存数据结构

定时内部状态扫描：
* GC
* chunkserver失效，chunk镜像
* 负载均衡，chunk镜像

每个chunk(64M)对应的元数据小于64byte，文件的namespace使用前缀压缩，通常也小于64byte。对应比例：1:10000，即16GB内存可以支持160TB的数据。

#### 2.6.2 chunk的位置

启动时从chunkserver获得所需的信息，在此之后，由于所有的修改操作都需经过master进行，因此master上的信息必定是最新的。

#### 2.6.3 操作记录

类似于Mysql的增量备份模式，checkpoint是一个类似B-树的格式，可以直接映射到内存。

### 2.7 一致性模型

一致性：经过修改后，clients看到的数据都是相同的

确定性：经过修改后，clients看到的数据都是相同的，并且能够知道每次修改的内容

并行写时，修改的内容以碎片的形式夹杂在一起，因此是不确定的
