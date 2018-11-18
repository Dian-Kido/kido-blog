---
layout:     post
title:      "浅谈Redis事务"
date:       2018-11-17
author:     "Kido"
header-img: "talking_redis_transaction-bg.png"
tags:
    - Redis
    - 后端
    
---

**Redis的事务和传统关系数据库的事务并不相同**

#### 传统数据库事务过程
在关系型数据库中，我们开启事务并进行一系列的读写操作，最后，用户用户可以选择发送commit来确认之前的修改，或者发送rollback来放弃之前的修改。

```javascript
    Connection conn = ... //获取数据库连接
    conn.setAutoCommit(false); //开启事务
    try{
       //......执行增删改查sql
       //......执行增删改查sql
       conn.commit(); //提交事务
    }catch (Exception e) {
      conn.rollback();//事务回滚
    }finally{
       conn.close();//关闭链接
    }
```

#### Redis事务过程
Redis提供的事务是将多个命令打包，然后一次性、按照先进先出的顺序（FIFO）有序的执行。在执行过程中不会被打断*（在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中）*，当事务队列中的所以命令都被执行*（无论成功还是失败）*完毕之后，事务才会结束。

```javascript
    MULTI       //开始事务
    SET   ...   //命令1入队
    GET   ...   //命令1入队
    SADD  ...   //命令1入队
    ......
    EXEC        //执行事务(一次执行1.2.3...)
```

如上所述，Redis的事务以特殊命令MULTI开始，之后跟着用户传入的多个命令，最后以EXEC未结束。**由于这种简单的事务在EXEC命令被调用之前不会执行任何实际操作，所以用户将没有办法根据根据读到的数据来做决定。**这句话初次读有点难懂，下面我们通过实例来解释一下这句话的含义。


## 用法分析

我们拿MySQL(InnoDB)和Redis在同样的需求场景下的不同实现形式来分析

> 场景： 
>假设我们有三条记录，分别为A、B、C；
>在MySQL中ABC代表同一行记录的三个字段，初始值分别为a、b、c；
> 在Redis中ABC代表三个不同Key键，初始值分别为a、b、c； 我们实现这样一个功能：
>  1. 将A的值变成a2
>  2. 当B=b的情况下，将C变成c2；如果这条这行不成功，则1、2两条都不执行

我们先看在MySQL中的伪代码实现
```javascript
    Connection conn = ... //获取数据库连接
    conn.setAutoCommit(false); //开启事务
    try{
        SET A = a2;                //将A的值变成a2
        var b_value = SELECT (B);  //读B的值(修改频繁的高并发场景下，这个读是很有必要的)
        if(b_value == b){
            boolean result = (SET C = c2 WHERE B=b);  //将C变成c2
        }
        
        if(result){
            conn.commit(); //提交事务
        }else{
            conn.rollback();//事务回滚
        }
        
    }catch (Exception e) {
      conn.rollback();//事务回滚
    }finally{
       conn.close();//关闭链接
    }
```

 
首先我们来看一个**Redis错误的示范**，看看Redis的事务有何不同
```javascript
ValueOperations vo = redisTemplate.opsForValue();
        vo.set("A", "a");//初始化数据
        vo.set("B", "b");
        vo.set("C", "c");
        redisTemplate.setEnableTransactionSupport(true);

        redisTemplate.multi();     //开始事务
        vo.set("A", "a2");         //将A的值变成a2
        String b_value = String.valueOf(vo.get("B"));    //获取B的值
        logger.info("事务读到b_value：" + b_value);
        if ("b".equals(b_value)) {
            vo.set("C", "c2");
        }else
        logger.info("事务结果：" + JSON.toJSONString(redisTemplate.exec()));
```
我们来看一下打印结果

    事务读到b_value：null
    事务结果：[true,"b"]

**看出Redis事务和传统数据库的区别了吗？**

我们可以看到第9行代码的返回结果是null，所以第12行根本就没机会执行。
我们从14行的打印结果返现有两个返回值：[true,"b"]，true是第8行vo.set("A", "a2")的返回值，"b"是第9行(vo.get("B")的返回值，也就是说redis这些命令，只有才EXEC执行的时候才会真正的执行。这就是之前文章提到的“**由于这种简单的事务在EXEC命令被调用之前不会执行任何实际操作，所以用户将没有办法根据根据读到的数据来做决定。**”，所以在EXEC执行前我们的普通的读写操作都不会执行的，在这个过程中我们不能像传统数据库那样读DB的值来做逻辑处理。

而且还有很重要的一点，**Redis是不支持已被执行命令的回滚**。

**那么在Redis中，面对这种场景我们应该如何处理呢?怎么解决读判断，以及判断回滚呢？**

那就要引入另一个关键字**watch**，我们先看怎么使用，再分析是怎么实现的
```javascript
try {
            redisTemplate.watch("B");//监视 Key-B
            String b_value = String.valueOf(vo.get("B"));    //获取B的值
            logger.error("事务读到b_value：" + b_value);
            if ("b".equals(b_value)) {
                redisTemplate.multi();     //开始事务
                vo.set("A", "a2");         //将A的值变成a2
                vo.set("C", "c2");         //将C的值变成c2
                logger.error("事务结果：" + JSON.toJSONString(redisTemplate.exec()));
            } else {
                redisTemplate.unwatch();
                logger.error("解除watch...");
            }
        } catch (Exception e) {
            redisTemplate.discard();//取消事务，放弃执行事务块内的所有命令
            logger.error("取消事务，放弃执行事务块内的所有命令");
        }
```

我们看一下如果在事务从“命令入队 ~ EXEC”期间，B键的值没有被其他客户端改变的情况下的打印结果：

    事务读到b_value：b
    事务结果：[true,true]
    
从打印结果，我们可以看到事务中7，8行的命令都被准确执行。

我们再Debug运行一次，这次我们把断点打在第8行，运行到第8行时我们在终端修改变B键的值：

> 127.0.0.1:6379> SET B b1
OK

这个时候我们让程序继续从第8行继续运行，打印结果为：

    事务读到b_value：b
    事务结果：[]

可以看到事务中7，8行的命令都没有执行，符合我们的预期。

## 具体实现

我们上面提到，Redis事务的过程分为三个阶段：

    1. 事务开始
    2. 命令入队
    3. 事务执行

#### 事务开始
Redis事务的开始是通过执行MULTI 命令来实现，它的作用是将执行该命令的客户端从非事务状态切换至事务状态， 这一切换是通过在客户端状态的 flags 属性中打开 REDIS_MULTI 标识来完成的。

#### 命令入队
当一个客户端出于事务状态时， 如果客户端发送的命令是 **EXEC（执行所有事务块内的命令） 、DISCARD（取消事务，放弃执行事务块内的所有命令。） 、 WATCH（监视任意数量的key ，提一下，在事务中执行这个命令会报错：ERR WATCH inside MULTI is not allowed） 、 MULTI（标记一个事务块的开始） 四个命令以外**的其他命令，那么服务器并不立即执行这个命令，而是将这个命令放入一个事务队列里面， 然后向客户端返回 QUEUED 回复。

#### 事务执行
当一个处于事务状态的客户端向服务器发送 EXEC 命令时， 这个 EXEC 命令将立即被服务器执行： 服务器会遍历这个客户端的事务队列，执行队列中保存的所有命令，最后将执行命令所得的结果全部返回给客户端。
**（这里需要说明的一点是，Redis在处理网络请求的是单线程的，所以队列中的命令在执行期间是不会被其他客户端命令插进来的。这一点对理解Redis事务很关键）**


#### WATCH
WATCH 用于事务开启之前对任意数量的Key进行监视，如果这个被监视的key被改动（这里提一下，**这个改动，不管是删除、添加、修改，或者A -> B -> A改回原值，都会被认为发生了改动**），那么相应事务就被取消，否则事务正常执行。所以我们可以认为 WATCH 是一个乐观锁。
如果想让key取消被监控，可以用 UNWATCH 命令(这里又要提一下，**UNWATCH 如果在事务中执行，也是会被放到队列里的**)。

如上文所说，**WATCH对于添加操作也是敏感的**，也就说我们可以对一个不存在的Key进行WATCH。假如cli_1客户端对一个不存在的Key进行WATCH，此时cli_2客户端增加了这个键，那么cli_1客户端的事务依然不会执行，这个很有意思



### 后记
本应该对redis事务相关的数据结构等展开来讲，但是篇幅有限（主要是能力有限，嘘~），所以想了解的可以去好好看看《Redis设计与实现》

### 参考文献
1.[《Redis设计与实现》黄健宏 著](http://redisbook.com/index.html)
2.[《Redis 实战》Josiah L. Carlson](https://book.douban.com/subject/26612779/)