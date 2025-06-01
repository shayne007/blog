---
title: Kafka In Action
date: 2021-05-17 13:12:17
updated: 2021-05-17 13:19:29
tags: [middleware, kafka]
categories: [kafka]
---

# 消息引擎系统

![Kafka Logo](https://uploader.shimo.im/f/GwdoPlAJ7tC6GGpA.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

* Apache Kafka 是一款开源的消息引擎系统。
* 消息引擎系统是一组规范。企业利用这组规范在不同系统之间传递语义准确的消息，实现松耦合的异步式数据传递。
* Kafka 是消息引擎系统，也是分布式流处理平台。

# Kafka 架构

![Kafka Architecture](https://uploader.shimo.im/f/sNIPG7KWTDuxoecr.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

设计目标：
* 提供一套 API 实现生产者和消费者；
* 降低网络传输和磁盘存储开销；
* 实现高伸缩性架构。

# Kafka 版本号

* 0.7 版本:只有基础消息队列功能，无副本；打死也不使用
* 0.8 版本:增加了副本机制，新的 producer API；建议使用 0.8.2.2 版本；不建议使用 0.8.2.0 之后的 producer API
* 0.9 版本:增加权限和认证，新的 consumer API，Kafka Connect 功能；不建议使用 consumer API；
* 0.10 版本:引入 Kafka Streams 功能，bug 修复；建议版本**0.10.2.2**；建议使用新版 consumer API
* 0.11 版本:producer API 幂等，事务 API，消息格式重构；建议版本 0.11.0.3；谨慎对待消息格式变化
* 1.0 和 2.0 版本:Kafka Streams 改进；建议版本 2.0；

# Kafka 的基本使用

## 如何做 kafka 线上集群部署方案？

https://time.geekbang.org/column/article/101107

## 集群参数配置

* Broker 端参数

```properties
# A comma separated list of directories under which to store log files
log.dirs=/home/kafka1,/home/kafka2,/home/kafka3
# Zookeeper connection string
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka1
# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=18000
# The address the socket server listens on
listeners=PLAINTEXT://:9092
# Hostname and port the broker will advertise
advertised.listeners=PLAINTEXT://:9092
# Log retention settings
log.retention.hours=168
log.retention.ms=15552000000
log.retention.bytes=1073741824
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
```

* Topic 参数

创建 Topic 时进行设置：

```bash
bin/kafka-topics.sh \
--bootstrap-server localhost:9092 \
--create \
--topic transaction \
--partitions 1 \
--replication-factor 1 \
--config retention.ms=15552000000 \
--config max.message.bytes=5242880
```

修改 Topic 时设置：

```bash
bin/kafka-configs.sh \
--zookeeper localhost:2181 \
--entity-type topics \
--entity-name transaction \
--alter \
--add-config max.message.bytes=10485760
```

* JVM 端参数

```bash
export KAFKA_HEAP_OPTS="--Xms6g --Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS=" \
-server \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=20 \
-XX:InitiatingHeapOccupancyPercent=35 \
-XX:+ExplicitGCInvokesConcurrent \
-Djava.awt.headless=true"
```

## 无消息丢失配置

1. 不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。
2. 设置 acks = all。acks 是 Producer 的一个参数，代表了你对"已提交"消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是"已提交"。这是最高等级的"已提交"定义。
3. 设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。
4. 设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。
5. 设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
6. 设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是"已提交"。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
7. 确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。
8. 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。

## 生产者分区策略

## 数据可靠性保证

## 消息幂等与事务

## 消费者组

![Consumer Group](https://uploader.shimo.im/f/jsSjwxqokB6rcxVp.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

**rebalance**

* session.timeout.ms
* heartbeat.interval.ms

要保证 Consumer 实例在被判定为"dead"之前，能够发送至少 3 轮的心跳请求，即 session.timeout.ms >= 3 * heartbeat.interval.ms。

* max.poll.interval.ms

This is a safety mechanism which guarantees that only active members of the group are able to commit offsets. So to stay in the group, you must continue to call poll.

* GC 参数

The recommended way to handle these cases is to move message processing to another thread, which allows the consumer to continue calling`poll`while the processor is still working. Some care must be taken to ensure that committed offsets do not get ahead of the actual position.

## 位移提交

https://time.geekbang.org/column/article/106904

![Offset Commit](https://uploader.shimo.im/f/1aQ3bQwlOaMxZ7zE.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

* 自动提交

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

一旦设置了 enable.auto.commit 为 true，Kafka 会保证在开始调用 poll 方法时，提交上次 poll 返回的所有消息。从顺序上来说，poll 方法的逻辑是先提交上一批消息的位移，再处理下一批消息，因此它能保证不出现**消费丢失**的情况。但自动提交位移的一个问题在于，它可能会出现**重复消费**。  
重复消费发生在 consumer 故障重启后，重新从磁盘读取 commited offset。只要 consumer 没有重启，不会发生重复消费，因为在运行过程中 consumer 会记录已获取的消息位移。

* 手动提交

```java
// 同步阻塞
consumer.commitSync()
// 异步回调
consumer.commitAsync(callback)
```

可同时使用 commitSync() 和 commitAsync()。对于常规性、阶段性的手动提交，我们调用 commitAsync() 避免程序阻塞，而在 Consumer 要关闭前，我们调用 commitSync() 方法执行同步阻塞式的位移提交，以确保 Consumer 关闭前能够保存正确的位移数据。将两者结合后，我们既实现了异步无阻塞式的位移管理，也确保了 Consumer 位移的正确性。  
比如我程序运行期间有多次异步提交没有成功，比如 101 的 offset 和 201 的 offset 没有提交成功，程序关闭的时候 501 的 offset 提交成功了，就代表前面 500 条还是消费成功了，只要最新的位移提交成功，就代表之前的消息都提交成功了。

## 消费者组消费进度监控

* 使用 Kafka 自带的命令行工具 kafka-consumer-groups 脚本。

```bash
./kafka-consumer-groups.sh \
--bootstrap-server kafka:9092 \
--describe \
--all-groups
```

![Consumer Groups](https://uploader.shimo.im/f/Cs83PebnjYOZFDXU.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

* 使用 Kafka Java Consumer API 编程。
* 使用 Kafka 自带的 JMX 监控指标。

# Kafka 的副本机制

# Kafka 请求 Reactor 处理机制

broker 参数

```properties
# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3
# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8
```

![Reactor Pattern](https://uploader.shimo.im/f/lDRma97OpwXksWpD.png!thumbnail?fileGuid=Ggk3HpKgtRQH3Dh6)

# 高水位和 Leader Epoch 机制

https://time.geekbang.org/column/article/112118

# 管理与监控

## 主题日常管理

```bash
# 创建主题
bin/kafka-topics.sh \
--bootstrap-server broker_host:port \
--create \
--topic my_topic_name \
--partitions 1 \
--replication-factor 1

# 查看主题列表
bin/kafka-topics.sh \
--bootstrap-server broker_host:port \
--list

# 查看主题详情
bin/kafka-topics.sh \
--bootstrap-server broker_host:port \
--describe \
--topic <topic_name>

# 修改分区数
bin/kafka-topics.sh \
--bootstrap-server broker_host:port \
--alter \
--topic <topic_name> \
--partitions <新分区数>

# 修改主题配置
bin/kafka-configs.sh \
--zookeeper zookeeper_host:port \
--entity-type topics \
--entity-name <topic_name> \
--alter \
--add-config max.message.bytes=10485760

# 删除主题
bin/kafka-topics.sh \
--bootstrap-server broker_host:port \
--delete \
--topic <topic_name>
```

## 动态参数配置

## kafka 调优

https://time.geekbang.org/column/article/128184

OS tuning

* Virtual Memory

```bash
# 查看当前配置
cat /proc/sys/vm/swappiness
cat /proc/sys/vm/dirty_background_ratio

# 修改配置
vi /etc/sysctl.conf
# The percentage of how likely the VM subsystem is to use swap space rather than dropping pages from the page cache.
vm.swappiness=1
# The percentage of the total amount of system memory, and setting this value to 5 is appropriate in many situations.
vm.dirty_background_ratio=5
```

```bash
# 查看当前状态
cat /proc/vmstat | egrep "dirty|writeback"
nr_dirty 11
nr_writeback 0
nr_writeback_temp 0
nr_dirty_threshold 67635
nr_dirty_background_threshold 33776
```

* Disk

```bash
mount -o noatime
``` 