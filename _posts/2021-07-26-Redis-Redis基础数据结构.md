---
layout: post
title: Redis-基础数据结构
categories: Redis
description: 介绍了Redis中五种
keywords: Redis
---

本文主要为《Redis深度历险》的读书笔记，以及对网上一些资料的摘录。可以在「[Redis 官⽅在线调试器地址](https://try.redis.io/)」上实践学习Redis的相关指令。

## string 

### 简介

​	Redis中的字符串是⼀种 动态字符串，使⽤者可以对其进⾏更改，底层类似于Java中的 ArrayList。当字 符串⻓度⼩于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时⼀次只会多扩 1M 的空间（最⼤ 为512M）。 

​	Redis 中 所有 的数据结构都是以 唯⼀ 的 key字符串 作为名称，然后通过这个唯⼀ key 值来获取相应的 value 数据。不同类型的数据结构的差异就在于 value 的结构 不⼀样。

​	字符串结构使⽤⾮常⼴泛，⼀个常⻅的⽤途就是缓存⽤户信息。我们将⽤户信息结构体使⽤ JSON 序列 化成字符串，然后将序列化后的字符串塞进 Redis 来缓存。同样，取⽤户信息会经过⼀次反序列化的过程。 



### 基本操作

**键值对** 

```shell
set [key] [val] # 存储键值对
get [key] 		# 按取键取值
exists [key] 	# 判断是否存在键
del [key] 		# 删除键值对
```



**批量键值对**: 可以批量对多个字符串进⾏读写，节省⽹络耗时开销

```shell
mset [key_1] [val_1] [key_2] [val_2] (...) 	# 批量存储键值对
mget [key_1] [key_2] [key_3] (...) 			# 返回⼀个列表
```



**过期和set命令扩展**: 可以对 key 设置过期时间，到点⾃动删除，这个功能常⽤来控制缓存的失效时间

```shell
expire [key] [t] 		# 设置 key t秒后过期
setex [key] [t] [val] 	# 存储键值对，同时设置过期时间
setnx [key] [val] 		# 若 key 不存在，则执⾏ set
```



**计数**:  如果 value 值是⼀个整数，还可以对它进⾏⾃增操作。⾃增是有范围的，它的范围是signed long 的最⼤最 ⼩值，超过了这个值，Redis 会报错。

```shell
incr [key] 			# 将 key 对应的 val + 1
incrby [key] [num] 	# 将 key 对应的 val + num
```

## list 

### 简介

​	Redis 的列表相当于 Java 语⾔⾥⾯的 LinkedList，插⼊删除操作时间复杂度为O(1)，索引定位的时间复 杂度为O(n)。

​	当列表弹出了最后⼀个元素之后，该数据结构⾃动被删除，内存被回收。

​	Redis 的列表结构常⽤来做异步队列使⽤。将需要延后处理的任务结构体序列化成字符串塞进 Redis 的 列表，另⼀个线程从这个列表中轮询数据进⾏处理。



### 基本操作

**当作队列使用**

```shell
rpush [key] [val_1] [val_2] [val_3] (...) 	# 创建名为 [key] 的 list
llen [key] 									# 查看 list [key] 的⻓度
lpop [key]									# 从 list 左侧出队
```



**当作栈使用**

```shell
rpush [key] [val_1] [val_2] [val_3] (...) 	# 创建名为 [key] 的 list
rpop [key]
```



**慢操作**

```shell
lindex
lrange [key] [start] [end] 	# 获取从 [start] 开始，[end] 结束的所有元素：lrang [key] 0 -1 获取所有元素
ltrim [key] [start] [end] 	# 保留 [key] 中 [start] ~ [end] 范围内的元素(闭区间)
lrem [key] [max_num] [val]	# 删除 [key] 中 “值=[val]” 的元素，最多删除 [max_num]个
lset [key] [index] [val] 	# 将 [key] 中索引为 [index] 的值，更改为val
```



### 快速列表

​	因为普通的链表需要的附加指针空间太⼤，会⽐较浪费空间，⽽且会加重内存的碎⽚化。Redis 底层存储 的不是⼀个简单的 linkedlist，⽽是称之为快速链表 quicklist 的⼀个结构。⾸先在列表元素较少的情况下会 使⽤⼀块连续的内存存储，这个结构是 ziplist，也即是压缩列表。它将所有的元素紧挨着⼀起存储，分配的 是⼀块连续的内存。当数据量⽐较多的时候才会改成 quicklist（也就是将多个 ziplist 使⽤双向指针串起来 使⽤）。



## hash 

### 简介 

​	Redis 的 hash 相当于 Java 语⾔⾥⾯的 HashMap，采⽤ 数组 + 链表 的数据结构。

**不同的是：**

- Redis中字典的值只能是字符串形式;

- Java 中的 HashMap 在字典很⼤时，rehash 是 个耗时的操作，需要 ⼀次性全部 rehash。Redis为了⾼性能，不能堵塞服务，所以采⽤了 渐进式 rehash 策略;

- Redis中的字典除了能扩容，还能缩容.

> **渐进式 rehash**
>
> 渐进式rehash，会在 rehash 的同时，保留新旧两个 hash 结构。查询时会同时查询两个hash 结构， 然后在后续的 定时任务 中以及 hash 的⼦指令中，循序渐进地将旧 hash 的内容⼀点点迁移到新的 hash 结构中。



## 基本操作

**添加元素**

```shell
hset [map] [key] [val] 								# 向map中⼀次增加⼀个
hmset [map] [key_1] [val_1] [key_2] [val_2] ... 	# 向map中批量增加元素
```



**获取 键\值**

```shell
hget [map] [key] 					# 获取key 对应的 val
hmget [map] [key_1] [key_2] ... 	# 批量获取 keys 对应的 vals
hgetall [map] 						# 获取 entries
hlen [map] 							# 获取 map 中 entry 的数量
hkeys [map] 						# 只获取 keys
hvals [map] 						# 只获取 vals
```



**删除元素**

```shell
hdel [map] [key_1] [key_2] ... 		# 批量删除和逐⼀删除 ⽤的是同⼀个指令
```



**判断元素是否存在**

```shell
hexists [map] [key] 				# 判断map中是否存在 key
```



**作为计数器的使用方式**

```shell
hincrby [map] [key] [num] 			# 将 key 对应的 count + num，(count默认从0开始)，若val 不是整数会报错
```



​	相比于字符串，hash 结构能够对结构中的每个字段单独存储，这样做可以进⾏信息的部分获取，节省⽹ 络流量。但是，hash 结构的存储消耗要⾼于单个字符串。



## set 

### 简介

​	Redis 中的 set 相当于 Java 中的 HashSet。它的内部实现相当于⼀个特殊的字典，字典中所有的 value 都是⼀个值 NULL。

### 基本操作

**添加元素**

```shell
sadd [map] [val_1] [val_2] ... # 批量和单个操作使⽤同⼀个指令
```



**获取元素**

```shell
smembers [set] 				# 列出所有元素
srandmember [set] [count] 	# 获取随机count个元素，如果不提供count参数，默认为1
scard [set] 				# 获取当前 set 中的元素总数
```



**删除元素**

```shell
spop [set] 						# 随机删除⼀个元素
srem [set] [val_1] [val_2] ... 	# 删除⼀到多个元素
```



**判断元素是否存在** 

```shell
sismember [set] [val] 			# 判断 set 中是否存在 val
```



## zset 

### 简介 

​	zset 是⼀个有序列表，底层实现使⽤了两个数据结构：第⼀个是**hash**，第⼆个是**跳跃列表**。hash的作⽤ 就是关联元素value和权重score，保障元素value的唯⼀性，可以通过元素value找到相应的score值；**跳跃列表**的⽬的在于给元素value排序，根据score的范围获取元素列表。 

### 基本操作

**增加元素**

```shell
zadd [zset] [score_1] [val_1] ... # 添加⼀个或多个元素
```



**获取排名和分数**

```shell
zscore [zset] [val] 	# 获取 val 对应的分数
zrank [zset] [val] 		# 获取 val 分数从低到⾼的排名(从0开始计数)
zrevrank [zset] [val] 	# 获取 val 分数从⾼到低的排名
```



**获取范围内的元素列表**

```shell
# 名次范围
zrange [zset] [start] [end] # 查询排名在 [strat, end] 之间的元素
zrange [zset] [start] [end] withscores # 查询的同时输出对应的分数
zrevrange [zset] [start] [end] 

# 分数范围
zrangebyscore [zset] [start] [end] # 查询分数在 [start, end] 之间的元素. inf 代表无穷大
zrangebyscore [zset] [start] [end] withscores
zrevrangebyscore [zset] [end] [start] # 注意：end 和 start 的次序反了⼀下
```



**删除元素**

```shell
zrem [zset] [val_1] [val_2] ... 		# 删除⼀个或多个元素
zremrangebyrank [zset] [start] [end] 	# 删除排名在 [strat, end] 之间的元素
zremrangebyscore [zset] [start] [end] 	# 删除分数在 [start, end] 之间的元素
```



## 容器型数据结构的通⽤规则

### create if not exists 

​	如果容器不存在，那就创建⼀个，再进⾏操作。 

### drop if no elements 

​	如果容器⾥元素没有了，那么⽴即删除元素，释放内存。