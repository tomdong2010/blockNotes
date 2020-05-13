# redis

### Redis:

架构：单机，主从，集群

### 应用：
```
  1—缓存、持久化

         2—订阅、发布（消息队列、消息通知）

         3—分布式锁

         4—分布式Session共享  
```
### Redis简介
redis全称REmote DIctionary Server，是一个由Salvatore Sanfilippo写的高性能key-value存储系统，其完全开源免费，遵守BSD协议。Redis与其他key-value缓存产品（如memcache）有以下几个特点。 
+ Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。 
+ Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。 
+ Redis支持数据的备份，即master-slave模式的数据备份。

Redis的性能极高且拥有丰富的数据类型，同时，Redis所有操作都是原子性的，也支持对几个操作合并后原子性的执行。另外，Redis有丰富的扩展特性，它支持publish/subscribe, 通知,key 过期等等特性。

