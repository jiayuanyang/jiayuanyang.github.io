+++
title = "InnoDB加锁分析"
date = "2024-06-12T00:00:00+08:00"
tags = ["MySQL", "锁", "隔离级别"]
description = "数据库加锁分析 隔离级别"

+++

# 加锁规则

- MySQL实战45讲总结的规则(5.x版本，规则可能会变化)

两个“原则”、两个“优化”和一个“bug”：

原则 1：加锁的基本单位是 next-key lock。希望你还记得，next-key lock 是前开后闭区间。

原则 2：查找过程中访问到的对象才会加锁。

优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。

优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。

一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。



- 其他参考

[不同语句加锁](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)

MySQL8 performance_schema.data_locks;



# InnoDB 等值查询加锁分析

仅分析等值查询，范围查询比较复杂

- 索引：唯一索引/非唯一索引
- 隔离级别：RR/RC
- 是否有符合条件的行：是/否




## 唯一索引

表结构

```
create table t (
 id int not null,
 uid int not null,
 val int not null,
 primary key (id),
 unique key uk_uid (uid)
) engine = innodb;

insert into t values (5, 5, 5), (10, 10, 10), (15, 15, 15), (20, 20, 20);
```

### RR隔离级别

#### 命中行

`begin; select * from t where uid = 5 for update;`

仅对记录5加锁

#### 未命中行

session1 `begin; select * from t where uid = 13 for update;`
锁(10, 15) 的gap，没有行锁


### RC隔离级别

#### 命中行

`begin; select * from t where uid = 5 for update;`

仅对记录5加锁

#### 未命中行

session1 `begin; select * from t where uid = 13 for update;`

session1 没有gap lock，没有行锁

session2 可以插入任意间隙，可以更新已有记录



## 非唯一索引

表结构

```
create table t (
 id int not null,
 uid int not null,
 val int not null,
 primary key (id),
 key idx_uid (uid)
) engine = innodb;

insert into t values (5, 5, 5), (10, 10, 10), (15, 15, 15), (20, 20, 20), (24, 20, 20), (28, 20, 20), (40, 40, 40);
```

### RR隔离级别

#### 命中行

session 1 `begin; select * from t where uid = 20 for update;`

锁(20, 40) 的gap								

3个uid=20的行锁

#### 未命中行

session 1 `begin; select * from t where uid = 18 for update;`

锁(15, 20(id=20))的 gap，可以插入(21, 20, 20)，不能插入(19, 20, 20)，注意非唯一索引，同一个索引值之间的间隙与主键id相关

uid=20(id=20)不需要加行锁

### RC隔离级别

#### 命中行

session 1 `begin; select * from t where uid = 20 for update;`

只加uid=20行锁

#### 未命中行

session 1 `begin; select * from t where uid = 18 for update;`

不加锁




## 结论

RR级别需要加gap lock，gap lock加record lock称作next-key lock

RC不需要加gap lock

- RR隔离级别（等值查询）
  - 唯一索引
    - 命中时，锁命中的一条记录
    - 未命中，锁间隙，下一条记录无需加行锁
  - 非唯一索引
    - 命中时，锁间隙，锁符合条件的记录行锁
    - 未命中时，锁间隙，下一条无需加行锁

- RC隔离级别（等值查询）
  - 唯一索引
    - 命中时，锁命中的一条记录
    - 未命中时，无需加锁
  - 非唯一索引
    - 命中时，锁命中的记录
    - 未命中时，无需加锁



## 如何验证锁相关问题

- InnoDB update未命中索引时，RR级别是否锁全表？
  - 实际上是扫描主键，逐行加锁
  - 验证：session 1，先锁住最后一个主键，session 2 进行更新，再对最后一个主键加锁时，需要等待，session 1更新第一个主键，会报死锁错误
    - 说明未命中索引的更新并不是先加表锁，而是先加了行锁
- RC级别，会提前释放不满足条件的行，是在语句执行完释放，还是在判断不满足条件后理解释放？
  - 验证：同样可以session1锁住最后一行，session 2进行更新，where条件有不符合条件的，当阻塞在最后一行时，session 3更新被session 2释放的行



# 怎么用锁实现各隔离级别

1. 读未提交
   1. 读不加锁，写加写锁
      1. 读不加锁，但是也能保证数据的正确。例如varchar(20)，不加锁也不会读错
2. 读已提交
   1. 读加读锁，读完释放，写加写锁
3. 可重复读
   1. 读加读锁，事务提交释放，写加写锁
4. 串行化
   1. 读写都加锁



举例，有并发事务t1, t2

- 读未提交，读不加锁，t2更新了一行， 还未提交，t1就可以读到，t2回滚，t1脏读

- 读已提交，读加读锁，t2更新一行，未提交时，t2持有写锁，t1读不到，t2提交后，t1加上读锁，可以读到

- 可重复读，t1读锁提交时释放，期间t2无法修改t1读到的记录，自然实现可重复读，但是仍然会有幻读，t2可以插入新记录
- 串行化，都加锁，不会出现幻读



## MVCC实现

只用锁也能实现各个隔离级别，但是并发较低

写加写锁，大部分无法进行读

MVCC支持写时读快照版本，增加并发