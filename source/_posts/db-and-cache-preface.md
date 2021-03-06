---
layout:     post
title:      "缓存与数据库一致性系列-序言"
date:       2018-11-24
author:     "Kido"
header-img: "db-and-cache-bg01.jpg"
tags:
    - 缓存
    
---

一般来说，对于一个新的业务，一般会经历这几个阶段：

### 阶段1：单库阶段
读写流量都比较小，这个时候所有的读写操作都在主库就ok了
这个时候，从库可能只是用来灾备
**风险分析：**从数据一致性角度来说没有风险，全走主库美滋滋~

### 阶段2：多库阶段
#### 阶段2.1：
单库扛不住了，这个时候就会考虑到分库分表了，通过增加数据库的方式，把单库的QPS降下来
**风险分析：** 从数据一致性角度来说没有风险，全走主库依然美滋滋~
#### 阶段2.2：
读流量增加，主库的读QPS偏高，这个时候我们就想着把从库得利用起来，于是读写分离：写主库，读从库
**风险分析：**读从库就意味着，**读到的数据可能不是最新的**，在实时性要求比较高的业务中是不能接受的，那应该如何解决呢？由于"DB主从一致性"不是本系列讨论的重点，所以这里推荐一篇文章,它介绍了几种比较不错的解决方案，感兴趣的大家可以去读一下：[DB主从一致性架构优化4种方法](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959442&idx=1&sn=feb8ff75385d8031386e120ef3535329&scene=21#wechat_redirect)

### 阶段3：数据库+缓存
对于互联网公司来说，大部分业务都面临着读多写少的情况（如果你写都有压力的话，那你做的业务一定很赚钱吧），而数据库资源是极其宝贵的，所以我们没有办法一味的添加从库，或者说不断的增加分库分表来解决，一是因为老板不会同意，二是不断分库分表的话，数据迁移过程也是高风险的。
所以，这个时候我们一般会通过添加的缓存的方式，来解决读压力
**风险分析：缓存与数据库数据一致性问题**

这就引出本系列文章，想要讨论的问题：**缓存与数据库数据一致性问题**，在提出解决方案之前，我们应该分析自己当前的业务和架构，看看自己的要达到的目标是什么？不同的方案解决的问题是不一样的

### 所以，你的业务和目标是怎样的呢？
 - 你是要求最终一致性、还是强一致性？
 - 你对缓存一致性的要求到底有多高？延迟1ms行不行？延迟1min行不行？
 - 你当前是单库单表？还是多库多表？
 - 你缓存的数据结构是怎样的？是否存在多表合并？是否存在多行合并的情况？甚至多库合并的情况？
 - 该如何容灾?更新、删除缓存失败你能不能接受？写数据库失败怎么办？
 - 如果删除缓存失败，你还允不允许更新数据库？
 - ......
 
只有我们看清楚自己的系统需要达到一种怎样的要求，才能针对性的制定对应的方案。
本系列文章，会针对不同的场景提出不同的解决方案，并从并发、容灾等各方面分析其优缺点

### 相关文章链接

 1. [《缓存与数据库一致性系列-01》](/2018/12/01/db-and-cache-01/)
 2. [《缓存与数据库一致性系列-02》](/2018/12/07/db-and-cache-02/)
 3. [《缓存与数据库一致性系列-03》](/2018/12/08/db-and-cache-03/)
 4. [《缓存与数据库一致性系列-04》](/2018/12/09/db-and-cache-04/)

### 后记
如果发现文章中有错误的地方，请联系我~
Mail：chen_dianshu@sina.com
