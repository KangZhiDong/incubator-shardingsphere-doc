+++
toc = true
title = "分布式主键"
weight = 2
+++

## 实现动机

传统数据库软件开发中，主键自动生成技术是基本需求。而各个数据库对于该需求也提供了相应的支持，比如MySQL的自增键，Oracle的自增序列等。
数据分片后，不同数据节点生成全局唯一主键是非常棘手的问题。同一个逻辑表内的不同实际表之间的自增键由于无法互相感知而产生重复主键。
虽然可通过约束自增主键初始值和步长的方式避免碰撞，但需引入额外的运维规则，使解决方案缺乏完整性和可扩展性。

目前有许多第三方解决方案可以完美解决这个问题，如UUID等依靠特定算法自生成不重复键，或者通过引入主键生成服务等。
但也正因为这种多样性导致了Sharding-Sphere如果强依赖于任何一种方案就会限制其自身的发展。

基于以上的原因，Sharding-Sphere最终采用以接口来实现对于生成主键的访问，而将底层具体的主键生成实现分离出来。

## 默认分布式主键生成器

采用snowflake算法实现，生成的数据为64bit的长整型数据。

其二进制表示形式包含四部分，从高位到低位分表为：1bit符号位(为0)，41bit时间位，10bit工作进程位，12bit序列位。

该算法保证不同进程的主键肯定是不同的，同一个进程首先是通过时间位保证不重复，如果时间相同则是通过序列位保证。
同时由于时间位是单调递增的，且各个服务器如果大体做了时间同步，那么生成的主键在分布式环境可以认为是总体有序的，这就保证了对索引字段的插入的高效性。例如MySQL的Innodb存储引擎的主键。

在数据库中应该用大于等于64bit的数字类型的字段来保存该值，比如在MySQL中应该使用BIGINT。

类名称：`io.shardingjdbc.core.keygen.DefaultKeyGenerator`

### 时间位(41bit)

从2016年11月1日零点到现在的毫秒数，时间可以使用到2156年，满足大部分系统的要求。

### 工作进程位(10bit)

该标志在Java进程内是唯一的，如果是分布式应用部署应保证每个进程的工作进程Id是不同的。该值默认为0，可通过调用静态方法`DefaultKeyGenerator.setWorkerId("xxxx")`设置。

### 序列位(12bit)

该序列是用来在同一个毫秒内生成不同的Id。如果在这个毫秒内生成的数量超过4096(2的12次方)，那么生成器会等待到下个毫秒继续生成。