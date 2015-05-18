# kafka学习笔记

## 什么是kafka

最近公司需要上基于nginx log的数据统计系统。其中一个重要的结点即分布式日志收集。在调研了多种方案之后，最终确定了flume+kafka+storm+hbase的系统架构。其中kafka则是linkedin一个专门为日志而产生的service。官方文档上如是说：Kafka是一个分布式、分区、冗余的commit日志service。它提供了一种特殊设计的消息系统功能。

## 特点

- 不支持事务
- 不保证全局消息顺序，可以保证partion消息顺序
- 顺序写磁盘，性能可媲美内存操作
- 无论消息是否被消费，都会持久化保存（保存时间可以设置）
- 消费者看到的消息顺序即是保存在log中的顺序
- 对于一个复制因子(replication factor)为N的topic,可以保证在N-1个server挂掉的情况下，已经提交到log中的消息不会丢失。

## 组成部分

总体结构如下图：

![](http://kafka.apache.org/images/producer_consumer.png)

- kafka将消息以category的方式保存在一起，称为topic
- 向topic产生消息的进程称为producer
- 处理topic上的消息的进程称为consumer
- kafka集群由一个或者多个server组成，称为broker.

### Topics and Logs