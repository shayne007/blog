---
title: ElasticSearch Notes for Interview
author: Charlie
date: 2025-05-28 16:51:37
tags: [elasticsearch]
categories: [elasticsearch]
---

* Es基础知识

* 关键词

   * 基于**Lucene**的Restful的分布式实时全文搜索引擎，快速存储、搜索、分析海量数据。

   * **Index**索引：存储数据的地方，类似mysql的表。

   * **document**文档：类似mysql种的一行数据，但是每个文档可以有不同的字段。

   * **field**字段：最小单位。

   * **shard**分片：数据切分为多个shard后可以分布在多台服务器，横向扩展，提升吞吐量和性能。

   * **replica**副本：一个shard可以创建多个replica副本，提供备用服务，提升搜索吞吐量和可用性。

   * **text**与**keyword**：是否分词。

   * **query**与**filter**：是否计算分值。


* Es的底层原理

* 关键词

   * **倒排索引**

   * **前缀树**，也叫做**字典树**（Trie Tree）

   * **refresh**操作

   * **flush**操作

   * **fsync**操作

* 什么倒排索引

>每个文档都有对应的文档ID，文档内容可以表示为一系列关键词（**Term**）的集合。通过倒排索引，可以记录每个关键词在文档中出现的次数和位置。
>倒排索引是关键词到文档ID、出现频率、位置的映射（**posting list**），每个关键词都对应一系列的文件，这些文件都出现了该关键词。
>每个字段是分散统计的，也就是说每个字段都有一个posting list。关键词的查找基于前缀树，也叫做字典树（trie tree），es里面叫做**Term Dictionary**。为了节省内存空间，es对前缀树做了优化，压缩了公共前缀、后缀，就是所谓的**FST（Finite State Transducers）**。
* 写入数据

**文档 -> Indexing Buffer -> Page Cache -> 磁盘**

**Translog -> Page Cache -> 磁盘**

   * refresh刷新操作将索引缓存写入到 Page Cache，保存为segment段文件，一个段里面包含了多个文档，refresh默认**1秒钟**执行一次，此时文档才能够被检索，这也是称Es为近实时搜索的原因。

   * Page Cache通过异步刷新（fsync）将数据写入到磁盘文件。

   * 文档写入缓存的时候，同时会记录Translog，默认**5秒钟**固定间隔时间刷新到磁盘。

* 前缀树


* Es怎么保证高可用？

* 关键词

   * Es高可用的核心是**shard**分片与**replica**副本

   * **TransLog**保障数据写入的高可用，避免掉电时的写入丢失

* Es高可用的基本保证

>Es高可用的核心是分片，并且每个分片都有主从之分，万一主分片崩溃了，还可以使用从分片，也就是副本分片，从而保证了最基本的可用性。
>Es在写入数据的过程中，为了保证高性能，都是写入到自己的Buffer里面，后面再刷新到磁盘上。所以为了降低数据丢失的风险，es还额外写了一个Translog，类似于Mysql里的redo log。后面es崩溃之后，可以利用Translog来恢复数据。
* Es高可用的额外优化

   * 限流保护节点：插件、网关或代理、客户端限流。

   * 利用消息队列削峰：数据写入数据库，监听binlog，发消息到MQ，消费消息并写入Es。

   * 保护协调节点：使用单一角色；分组与隔离。

   * 双集群部署：消息队列两个消费者双写到AB两个集群。


* Es查询性能优化？

* 关键词

   * jvm参数

   * 本地内存优化

* 高性能方案

   * 常规方案

      * 优化垃圾回收

      * 优化swap

      * 文件描述符

   * 优化分页查询

   * 批量提交

   * 调大refresh时间间隔

   * 优化不必要字段

   * 冷热分离

* JVM本地内存优化

场景：Elasticsearch 的 Lucene 索引占用大量堆外内存（Off-Heap），配置不当易引发 OOM。

优化方案：

   * 限制字段数据缓存（fielddata）大小，不超过堆内存的 30%

      * indices.fielddata.cache.size: 30% 

   * 优化分片（shard）数量

      * 单个分片大小建议在 10-50GB 之间，**过多分片会增加堆外内存开销**。

      * 例如：100GB 索引，分配 2-5 个分片。


* Es的实战应用

* [视频图片结构化分析系统](https://shimo.im/docs/PJtKVDqYkYXddjgJ)使用**ElasticSearch**存储结构化信息支持目标检索功能

   * 技术选型分析

      * 需要存储哪些数据：视频分析结果数据（人形、车辆、人脸、骑行等）存储。

      * 支持目标检索

   * 部署架构与高可用：三节点部署集群。

   * 性能优化：Es批量写入、Es使用Kafka异步写入、refresh时间间隔配置修改。

   * 常见问题解决经验

      * 数据丢失与备份

      * 分页查询（/image_result/_search?scroll=10m）

      * 脑裂问题

      * 中文分词

         * ik中文分词器

         * **定制分词器**：结巴分词+公安特征词库（10万+专用词汇）

            * ik分词器的配置文件中[修改远程扩展词典](https://zhuanlan.zhihu.com/p/468392276)


