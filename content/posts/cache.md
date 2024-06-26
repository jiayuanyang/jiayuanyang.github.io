+++
title = "缓存使用经验"
date = "2024-05-27T22:00:00+08:00"
tags = ["Redis", "并发编程", "时序", "数据一致性"]
description = "Redis 缓存更新的时序问题 数据一致性 主从延迟 zset更新时序"

+++

# 一、Redis

## 1.1 大key、热key问题

- 大key
  - 压缩：使用JSON序列化时，可以进行压缩，一般压缩率能在50%+
  - 拆分：假设使用string存浏览记录列表
    - 记录过多时，数据过大，需要拆分
    - 方式1：构造一个总key，history-summary:$uid，总key里保存分key信息
    - 方式2：history:$uid直接存第一页信息，额外存总数量，其余全部页的key。数量少时，访问一次即可
      - 下一页的key为$history_p1:$uid, hisotry_p2:$uid时，如果更新部分key失败，会有不一致问题
        - 分页可以使用随机key，在第一页里标明，给前端page token，下一次可以用token作为key取数据
        - 删除时，需要额外一个get操作， 先get到分页随机key，再删除
      - 分页的如果用hashtag，可以使用redis事务保证同时成功，但是会集中在一个节点
- 热key
  - 拆分：例如计数，拆成_0, _1，读取时合并
  - 复制：只读的key，可以复制到_0, _1, _2，读取时随机读一个， 更新时一起更新
    - 情况允许时，redis也可以读从库，但一般不会用
  - 本地缓存：热key可以在本地进行缓存
    - 实例较少时，本地热key发现即可
    - 实例较多时，分布式统计
  - 写热点：MQ削峰聚合
- 缓存穿透/雪崩
  - 缓存空哨兵
  - 随机过期时间
  - 布隆过滤器提前拦截



## 1.2 缓存/DB数据一致性

常见的缓存更新方式



- 先更新DB，后更新缓存
  - 更新缓存失败， 取到老数据
    - 如果是更新DB后异步更新缓存，多次缓存写的顺序不确定
    - 同步写，失败了还是老数据，还是要做重试，可能会有多个缓存更新顺序问题
  - 更新缓存不建议放到DB事务中（见到过线上这么处理的）
    - 开启事务，更新DB，更新缓存，更新缓存成功后提交事务，更新缓存失败，回滚事务
      - 首先，更新缓存失败，可能是超时实际缓存已更新， 此时回滚DB造成数据不一致
      - 更新缓存超时失败，再次查缓存查到了最新数据，此时commit，commit也可能失败，数据还是不一致
- ~~先更新缓存，后更新DB~~ 排除
  - 更新DB失败，缓存数据是错误的
- 先更新DB，后删除缓存
  - 删除缓存失败， 会导致取到老数据，可以进行重试删除。或者什么都不做，等待缓存过期
    - 即使删除成功，由于**DB主从延迟**，还是有可能读从库读到旧数据，写入缓存旧数据
      - 缓存也有主从延迟的问题， 但一般缓存不读从库，不必考虑缓存的主从延迟
    - 可以引入延迟删除，避免主从同步延迟
- 先删除缓存，后更新DB
  - 删除缓存和更新DB期间，会读到旧数据， 更新旧数据到缓存
    - 和上面类似



先更新缓存，后更新DB逻辑有问题

其余3种方式都大同小异，都没有100%完美



- 缓存操作不要放到DB事务
- 为避免主从延迟，可以采用延迟删除
- **高并发**时，删除缓存会导致大量请求到DB，尽量采用**更新缓存**方式
  - 更新缓存比删除缓存更复杂，有时要再拉取不同数据，组合起来
  - 更新缓存有2个结果：新的数据 或者旧的数据
    - 成功，新缓存
    - 失败/超时，旧缓存或新缓存 (不考虑缓存过期)
  - 而删除缓存的2个结果：旧的数据 或者无缓存， 这个结果更确定一些，重试无副作用， 而更新缓存的重试， 要考虑有没有**更新的数据更新**
    - 成功，缓存被删除
    - 失败/超时，旧缓存 或者 缓存被删除
- 更新缓存的时序会导致写入缓存顺序不一致，可以通过订阅binlog，串行更新



很多时候， 不必考虑太细， 我们认为**时间差**足够用应付上述的极端问题

- 先更新DB，后删除缓存
  - 有主从延迟问题， 但主从延迟一般很小，再次查询的时间差足够同步
- 先删除缓存，后更新DB
  - 删除缓存 与 更新DB的时间差内，可能将旧数据写入缓存，认为这个时间差很小，忽略
  - 再有，即使期间没有读操作，也仍有主从延迟问题



总结

数据库与缓存是两个系统，总会有不一致

**时间差**与**时序**会有引发各种问题



### 一种更新的方式，尽力避免主从延迟，避免不一致

例如uid的账号基本信息， 空哨兵可以存empty特殊字符，也可以存{uid:-1}， 用-1标记为空

参考这种方式

更新DB后， 写入缓存{uid: -2} （-2标识最近有更新，读取时需要再次读从库，或者主库）  过期时间 略大于主从延迟

应用读缓存，读到-2时，时效要求强的，可以读主库  (读了主库也不更新缓存)， 时效要求低的，读从库

等到-2过期后，缓存无数据，再次读从库，构造缓存



并发更新时，会一直向缓存写入-2，等到-2过期，认为最近没有更新，可以读从库构建数据



这个方式假设主从复制在一个固定时间内完成



### 另外一种更新方式

- 更新DB后，更新缓存

更新DB后，更新缓存期间，缓存可能过期

此时另外一个读请求可能读DB读到旧数据， 再更新DB请求，更新缓存之后， 写入缓存

更新DB后更新缓存，用SET写入

读请求构造缓存时，用SETNX写入

这样避免，读请求的老数据覆盖掉更新后的新数据



## 1.3 zset使用

用户点赞列表，zset score存时间戳，member存被点赞对象id

- 先读缓存，有数据直接使用
- 再读DB，写入zset
- 写操作时， 需要先判断zset是否存在， 不存在则不更新zset
  - zset存在，则zadd



简单实现就是上述方案，但是上述有时序问题

- 先读缓存，zset不存在
- 此时读DB，读了100条数据，还未写入zset
- 新来的写请求，新点赞记录，判断zset不存在，未写入zset
  - 如何判断zset存在？exist判断，ttl判断，可能恰好只剩下1s过期，不完美
  - 可以用expire 延迟过期时间， expire $key 60 更新成功返回1，key不存在返回0
- 此时100条数据写入zset， zset丢失了最新一条数据



读zset，不存在时，再读DB之前，应该向zset写入一个假数据， 例如score时间戳无限大/无限小， member=-1

这样，写请求就可以将最新的数据写入

读缓存时，要过滤掉member=-1





## 1.4 并发更新缓存的一些通用解法

### 缓存提供版本号set能力  cas更新

SET时传入版本号，版本号大于key的版本号时，才能写入



### 监听binlog更新
binlog有序，串行操作

合并：同一个key， update 1, 2, 3 可以合并后只处理最后一个写



### redis 的watch  multi

太麻烦



# 二、本地缓存

redis的性能很高，单节点能够达到10W QPS



但是有时候还是有瓶颈，可以再本地再做一层缓存

- 定期更新

- 异步更新



哪些key要本地缓存？

- 预先知道的，可以云控下发
- 未知的，热key发现
  - 本地记录
    - 可以用lfu+滑动窗口， 可以参考redis的lfu实现
    - 采样， 1/10 1/20，每10个/20个key 统计一次  100W qps，降到10w qps



- 读合并
  - golang的singleflight， 多个key合并为一次访问
    - 第一个请求的ctx过短，失败后，被合并的请求也立即失败
      - 可以加一些策略进行重试
  - singleflight也有一些问题：例如单个连接异常，被合并的请求都会被影响