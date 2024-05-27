+++
title = "InnoDB索引占用空间分析"
date = "2024-05-27T09:00:00+08:00"
tags = ["MySQL", "数据库", "存储", "慢查询"]
description = "索引空间可视化分析，慢查询，索引选择"

+++

# 背景

一两年前在申请Redis资源时，预估了N GB，上线后查看监控， 数据占用大于N GB（大很多）

因此阅读Redis源码，了解到内部一些数据结构的额外开销：dictEntry，robj，sds header，malloc碎片等等



对于MySQL，一直不确定空间在上有哪些开销

## 问题

新建如下表，插入100W行记录，占用多大空间？

```
create table test (
    id bigint not null auto_increment,
    uid bigint not null,
    status int not null,
    ctime bigint not null,
    mtime bigint not null,
    primary key (id)
) engine = innodb;
```

uid、status分别新增索引，会各自增加多大空间？

主键是bigint，一个page非叶子节点最多多少条记录？ 主键是int，又是多少



```
-- 存储过程插入100W行
DROP PROCEDURE IF EXISTS insert_batch;
DELIMITER $$
CREATE PROCEDURE insert_batch(max_num INT)
BEGIN
    DECLARE i INT DEFAULT 0;
    SET autocommit = 0;
    REPEAT
        INSERT INTO test (uid, status, ctime, mtime) VALUES (1, 2, 3, 4);
        SET i = i + 1;
        UNTIL i = max_num
    END REPEAT;
    COMMIT;
END$$
DELIMITER ;

call insert_batch(1000000);
```



## 分析工具

[innodb_ruby](https://github.com/jeremycole/innodb_ruby)      [作者博客](https://blog.jcole.us/innodb/)

ibd文件分析



# 一、索引空间占用

不同版本可能会有不同，仅供参考。下面测试使用MySQL 5.7版本



## 1.0 结论

### 聚簇索引

- 叶子节点
  - 各字段大小 +  18B（record header=5B + roll_ptr=7B + trx_id=6B）
- 非叶子节点
  - 主键大小 + 5B(record header) + 4B（指针）

### 二级索引
  - 叶子节点
    - 索引字段大小 + 主键大小 + 5B(record header)
- 非叶子节点
  - 索引字段大小 + 主键大小  + 5B(record header) + 4B（指针）



- 其他影响因素
  - [innodb_fill_factor](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_fill_factor) page内默认预留1/16空间
    
    - 测试验证时，叶子节点有预留，非叶子节点不预留
    
  - page directory：平均4条record一个slot，每个slot 2B，即1个记录0.5B
  
  - 每个page有128B的元数据开销
  
  - 页内碎片
  


[作者博客](https://blog.jcole.us/innodb/)里可以找到详细说明



### 计算

#### 聚簇索引

- 叶子节点

  - 记录大小 54B：各个字段占用 8(id)+8(uid)+8(ctime)+8(ctime)+4(status) = 36B + 额外开销18B=54B

  - 一页最多记录数：(16384 -128)*15/16 / 54.5 = 279.6330

- 非叶子节点

  - 记录大小： 8+5+4 = 17B
  - 一页最多记录数：(16384 -128) / 17.5 = 928.9143

#### 二级索引 idx_status为例

- 叶子节点
  - 记录大小：4+8+5 = 17
  - 一页最多记录数：(16384 -128)*15/16 / 17.5 = 870.8571
- 非叶子节点
  - 记录大小：4+8+5+4 = 21
  - 一页最多记录数：(16384 -128) / 21.5 = 756.0930



下面使用innodb_space分析验证



## 1.1 聚簇索引 叶子节点

### 叶子节点使用page数量 = 3585

![image-20240522232345900](/img/image-20240522232345900.png)

这里只分析INDEX页

ALLOCATED也会占用ibd空间，具体规则未深入研究




space-indexes显示 ibd文件中，叶子节点使用了3585个page (每个page 16KB)



### 叶子节点的记录数是139+204+279*3583

![image-20240523001012100](/img/image-20240523001012100.png)

index-level-summary   -l 表示第几层 -l 0为叶子节点

page4 139条记录，page3616 204条记录，剩余3583个page都是279条记录，139+204+279 * 3583 = 100W

一页最多279个记录，符合计算

page5， data=15066, records=279 15066/279=54，符合计算



free=1050 表示每页有1050剩余空间

1050/16384 = 0.0641 与 innodb_fill_factor 1/16=0.0625接近



### 每页的占用

![image-20240522234350316](/img/image-20240522234350316.png)

128B(页的结构信息) + 54 * 279（每条记录大小） * 140 (page directory开销) + 1050 (空闲) = 16384

一字节不差



### page directory 为什么是140字节

每个slot 2B

279 / 4 = 69.75，是向上进到70的吗？

![image-20240522235107637](/img/image-20240522235107637.png)

分析page5的page directory

可见并不是进上去的，确实有70个slot，其中infimum的owned固定为1 只包含自身，supremum为8，除自身外还有7个   4*68 + 7 = 279 刚好



## 1.2 聚簇索引 非叶子节点

![image-20240523001846807](/img/image-20240523001846807.png)

-l 1可以看到，每页最多928个记录，符合计算

15776/928 = 17，记录大小符合



## 1.3 二级索引 叶子节点

添加索引 alter table test add index idx_uid (uid);

![image-20240523002302195](/img/image-20240523002302195.png)

再添加一个索引  alter table test add index idx_status (status);

![image-20240523002320724](/img/image-20240523002320724.png)

idx_uid叶子大小应该为 8+8+5 = 21B

idx_status叶子节点大小应该为 4+8+5=17B



![image-20240523002706499](/img/image-20240523002706499.png)

idx_status举例

一页最多928个记录，符合计算

15776/928 = 17，符合计算



用叶子节点used进行校验

idx_uid.used/idx_status.used = 1325 / 1078 = 1.2291

21/17 = 1.2353 接近



## 1.4 二级索引 非叶子节点

idx_status非叶子节点大小  4+8+5+4 = 21



用-l 1的数据计算 

上面计算出最大记录数是756.0930，这里是755

15855/755=21 符合计算



## 1.5 总结

### 聚簇索引

叶子节点除了各字段外，还有roll_ptr，trx_id

### 二级索引

叶子节点没有roll_ptr，没有trx_id

叶子节点有主键id

非叶子节点也有主键id

ps：二级索引上没有roll_ptr、trx_id，MVCC可见性如何判断？

### 不同主键大小，非叶子节点一页最多存多少？

- bigint
  - 非叶子节点最多928.8 = (16384-128) /(8+5+4+0.5) 
  - 实际验证是928
- int
  - 非叶子节点最多1204.0 = (16384-128) /(8+5+4+0.5)  
  - 实际验证不是1204， 是1203
    - 与上面 计算得到756.0930，实际755类似， 实际上是能够再存一个记录，但是剩下一个空位



# 二、一些常见问题

- 为什么表不要超过千万行
  - bigint主键， 假设记录1k，叶子节点大概存15个
    - 两层非叶子节点928^2 = 861184 = 86.1W ，三层记录数1300W
    - 三层非叶子节点928^3=799178752  7.99亿 四层记录数 120亿
  - 三层时，前2层非叶子节点基本可以全缓存在内存，占用(928+1)*16K = 232.25MB
    - 增删改查仅需要访问一次磁盘
  - 四层时，不可能完全缓存前3层非叶子节点，需要访问2次磁盘
- 为什么索引快
  - 主键上字段很多，数据量大
  - 二级索引数据量小
  - 回表成本很高
- 分表分多少合适，以uid mod分表数为例
  - 首先，一定不是越多越好，分65536个，16384个表显然不合适
  - bigint主键， 完全缓存前两层非叶子节点占用14.5MB
  - 如果所有uid都是均匀访问
    - 分1024个表，前两层全在内存，占用14.5G，基本能够容纳
    - 分2048个表，想要全缓存前两层，需要29G内存， 比较大
  - 压测、实践验证为准



# 三、遇到的慢查询分析

explain



## 3.1 varchar字段 like %xxx% 不走索引

select * from t where country like'%US%' ; 用不了country索引

只能全表扫描



全表扫描一定比用索引快吗？

如果满足where的行比较少，只有少数回表，走索引会更快



force index (idx_country)， 是无法用上country索引的



需要使用子查询才能走country索引

select * from t where id in(select id from t where country like '%US%')

子查询里可以用到idx_country索引，只查id，不会走全表扫描



（这个场景后续改用了ES做查询）



## 3.2 覆盖索引

select count(*) from t where uid = 123 and status =  1; (uid有索引，(uid, status)没有索引)



大部分情况会用上uid索引，再回表判断status

有些uid数据较多，会走全表扫描

创建(uid, status)覆盖索引， 避免回表，就会选择索引了



覆盖索引，有时，仅仅为了覆盖，将原来多字段的聚簇索引抽出部分字段，组成一个“窄“表，加快查询

实际上就是空间换时间的体现



## 3.4 避免选择错误索引

select where uid = 123 order by ctime limit 10;



uid有索引，ctime有索引

我们更希望选择uid索引，但由于order by ctime，优化器有时会选择ctime



除了force index，还可以修改sql，order by (ctime + 0) 或者 order by ctime asc,  id desc (ctime asc, id asc 可以用索引)



## 3.5 深翻页

翻页时offset过大， 查询会变慢

因为被翻页的记录也都要回表



一种方法是不用offset，每次带上上次的id，where id > lastid 来查询

另外也可以用子查询，先查出id，再用id来查



## 3.6 离线worker扫表

需求是扫描ctime再(111, 999)范围，status是unpay的订单

select * from order_info where ctime > 111 and ctime < 999 and status = 'unpay' and id > $lastID order by id asc limit 1;

走了id索引，但是用主键扫描，首次定位第一个id，耗时较长



方法1：先用ctime索引，查出最大最小id  select min(id), max(id) from order_info where xxx

方法2：直接使用ctime索引，再回表，select * from order_info where (ctime = $lastTime and id > $lastID) or ctime > lastTime


