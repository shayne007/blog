---
title: MySQL Notes for Interview
date: 2025-06-01 16:40:22
tags: [mysql]
categories: [mysql]
---

* MySQL基础知识
* 关键词
   * 事务隔离级别、三范式
* 如何理解数据库表设计的三个范式
   * 第一范式：1NF 是对属性的原子性约束，要求**属性具有原子性**，不可再分解；
   * 第二范式：2NF 是对记录的惟一性约束，要求**记录有惟一标识**，即实体的惟一性；
   * 第三范式：3NF 是对字段冗余性的约束，即任何字段不能由其他字段派生出来，它要求**字段没有冗余**。
* 查询SQL的执行过程
   * 执行**连接器**
      * 管理连接，包括权限认证
   * 执行检索缓存（SQL语句与结果的kv存储）
   * 执行**分析器**
      * 词法分析
      * 语法分析
   * 执行**优化器**
      * 执行计划，选择索引方案
   * 执行**执行器**
      * 调用存储引擎接口
      * 表权限检查

* 数据库索引
* 关键词
   * B+树
   * 支持范围查询、减少磁盘IO、节约内存
* 为什么使用B+树
   * 与 B+ 树相比，**跳表**在极端情况下会退化为链表，平衡性差，而数据库查询需要一个可预期的查询时间，并且跳表需要更多的内存。
   * 与 B+ 树相比，**B树**的数据存储在全部节点中，对范围查询不友好。非叶子节点存储了数据，导致内存中难以放下全部非叶子节点。如果内存放不下非叶子节点，那么就意味着查询非叶子节点的时候都需要磁盘 IO。
   * 二叉树、红黑树等层次太深，大量磁盘IO。
   * **B+树的高度一般在2-4层（500万-1000万条记录），根节点常驻内存，查找某一键值的行记录时最多只需要1-3次磁盘IO。**
   * 通常使用**自增长的主键**作为索引
      * 自增主键是连续的，在插入数据的时候能减少**页分裂**，减少数据移动的频率。
* 索引失效的情况
   * 使用like、!=模糊查询
   * 数据区分度不大（性别等枚举字段）
   * 特殊表达式，数学运算和函数调用
   * 数据量小
* 最左匹配原则（本质上是由联合索引的结构决定的）
   * 索引下推：利用联合索引中数据检查是否满足where条件

* SQL优化
* 关键词
   * 执行计划是否使用索引
   * 索引列的选择
   * 分页查询的优化
* 查看执行计划
   * explain的字段含义
      * possible key、type、rows、extra等字段值的含义
      * 全表扫描考虑优化
* 索引列的选择
   * 外键
   * where中的列
   * order by的列，减少数据库排序消耗
   * 关联条件列
   * 区分度高的列
* 优化方案
   * 覆盖索引减少回表
   * 用where替换having（先过滤数据再分组，减少分组耗时）
   * 优化分页查询中的偏移量

* 数据库锁
* 关键词
   * 锁的种类、锁与索引
* 锁的分类
   * 根据锁的范围
      * 行锁
      * 间隙锁（左开右开），工作在可重复读隔离级别
      * 临键锁（左开右闭），工作在可重复读隔离级别
      * 表锁
   * 乐观锁、悲观锁
   * 互斥角度
      * 共享锁
      * 排他锁
   * 意向锁
* 锁与索引的关系
   * InnoDB的锁是通过索引实现的，锁住一行记录就是锁住用上的索引上的一个叶子节点，没有找到索引就锁住整个表

* MVCC协议
* 关键词
   * 版本链、读写操作
* 为什么需要MVCC
   * 避免读写阻塞问题
* 版本链
   * 事务id（trx_id）：事务版本号
   * 回滚指针(roll_ptr)
   ![回滚指针](https://uploader.shimo.im/f/pX4tjYjXIvdzRuhR.png!thumbnail?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NDg3NjgxNzksImZpbGVHVUlEIjoiWEtxNDJ5TE54ZUY5RWRBTiIsImlhdCI6MTc0ODc2Nzg3OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoiYWxsOmFsbDoiLCJ1c2VySWQiOjE4ODA2NDY4fQ.YXDluI8DZURAoXzZ-VQ2fc-bLD_FAb73Z1hN8_8umps)
   * undolog
      * 版本链存储咋在undolog，形似链表
* Read View
   * 不同的Read View，看到不同的活跃事务id列表（m_ids，未提交的事务）；
   * Read View与事务隔离级别
      * 已提交读：事务每次发起查询的时候，都会重新创建一个新的 Read View。
      * 可重复读：事务开始的时候，创建出 Read View，中间的多次读操作使用同一个Read View。

* 数据库事务
* 关键词
   * ACID
   * 隔离级别
* undolog
   * 用于事务回滚，存储了版本链
   * 具体内容
      * Insert操作，记录主键，回滚时根据主键删除记录
      * Delete操作，记录主键删除标记true，回滚时标记为false
      * Update操作
         * 更新主键，删除原记录、插入新记录
         * 没有更新主键，记录被更新字段原始内容
* redolog
   * 为什么需要redolog
      * 顺序写，性能好
   * redolog buffer刷盘
      * innodb_flush_log_at_trx_commit默认是1，事务提交时写入磁盘
* binlog
   * 二进制日志文件
   * 用途
      * 主从同步
      * 数据库出现故障时恢复数据
   * 刷盘（sync_binlog）
      * 0，默认值，由操作系统决定刷盘时机
      * N，每N次提交就刷盘，N越小性能越差
* 数据更新事务执行过程
   * 读取并锁住目标行到buffer pool
   * 写undo log回滚日志
   * 修改buffer pool中的数据
   * 写redo log
   * 提交事务，根据innodb_flush_log_at_trx_commit决定是否写入磁盘
   * 刷新buffer pool到磁盘（事务提交了，但buffer pool的数据不是立刻刷到磁盘）
   * 子流程：
      * 如果在 redo log 已经刷新到磁盘，然后数据库宕机了，buffer pool 丢失了修改，那么在 MySQL 重启之后就会回放这个 redo log，从而纠正数据库里的数据。
      * 如果都没有提交，中途回滚，就可以利用 undo log 去修复 buffer pool 和磁盘上的数据。因为有时，buffer pool 脏页会在事务提交前刷新磁盘，所以 undo log 也可以用来修复磁盘数据。

* 分库分表
* 关键词
   * 分治模式
   * 数量大时分表，并发高时分库
   * 分片算法
* 主键生成
   * 数据库自增主键，每个库设置不同的步长
   * 雪花算法
* 分片算法
   * 范围分片，时间范围等
   * hash取模分片
   * 一致性hash分片
   * 查表法
      * 分片映射表，映射关系可以根据流量动态调整
      * 分片映射表可以使用缓存，避免本身成为热点和性能瓶颈
* 分库分表的问题
   * join操作问题
   * count计数问题
   * 事务问题
   * 成本问题

