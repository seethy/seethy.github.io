---
title: HDFS架构
date: 2023-09-08 19:48:51
tags:
- 大数据
- HDFS
categories:
- ["组件服务", "HDFS"]
typora-root-url: ..
---

　　

​    ![HDFS架构图](/images/HDFS架构.jpeg)

　　HDFS 采用Master/Slave的架构来存储数据，这种架构主要由四个部分组成，分别为HDFS Client、NameNode、DataNode和Secondary NameNode。下面我们分别介绍这四个组成部分  

　　**1、Client：就是客户端。**

- 文件切分。文件上传 HDFS 的时候，Client 将文件切分成 一个一个的Block，然后进行存储。
- 与 NameNode 交互，获取文件的位置信息。
- 与 DataNode 交互，读取或者写入数据。
- Client 提供一些命令来管理 HDFS，比如启动或者关闭HDFS。
- Client 可以通过一些命令来访问 HDFS。

　　**2、NameNode：就是 master，它是一个主管、管理者。**

- 管理 HDFS 的名称空间
- 管理数据块（Block）映射信息
- 配置副本策略
- 处理客户端读写请求。

　　**3、DataNode：就是Slave。NameNode 下达命令，DataNode 执行实际的操作。**

- 存储实际的数据块。
- 执行数据块的读/写操作。

　　**4、Secondary NameNode：并非 NameNode 的热备。当NameNode 挂掉的时候，它并不能马上替换 NameNode 并提供服务。**

- 辅助 NameNode，分担其工作量。
- 定期合并 fsimage和fsedits，并推送给NameNode。
- 在紧急情况下，可辅助恢复 NameNode。
