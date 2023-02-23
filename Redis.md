# Redis笔记

### 概述

Redis（Remote Dictionary Server )，即远程字典服务，是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

#### Redis能干嘛？

1、内存存储、持久化，内存中的数据是断电即失的，所以持久化很重要（RDB、AOF）
2、效率高，可以用于高速缓存
3、发布订阅系统
4、地图信息分析
5、计时器、计数器（浏览量）
6、……

#### 特性

1、多样的数据类型
2、持久化
3、集群
4、事务
……

#### 基础知识

1、redis默认有16个数据库，默认使用的是第0个数据库，可以使用select切换数据库

```
select 1  #切换1数据库DBSIZE  #查看DB大小keys *  #查看所有keyflushall #清空数据库exists key #判断当前key是否存在move key 1 #移除当前的keyexpire key second #设置过期时间ttl key  #查看key剩余过期时间type key #查看key的类型
```

### Redis是单线程的！

官方表示，Redis是基于内存操作的，CPU不是Redis的性能瓶颈。Redis的瓶颈是根据机器的内存和网络带宽，既然可以使用单线程，就使用单线程了！

Redis是C语言写的，官方提供的数据为100000+的QPS，完全不比同样使用key-value的MemeCache差！

**Redis为什么单线程还这么快？**
误区1：高性能服务器一定是多线程的
误区2：多线程（CPU上下文切换）一定比单线程效率高
核心：redis是将全部数据放在内存中，所以使用单线程去操作效率就是最高的，对于内存系统来说，如果没有上下文切换效率就是最高的，多次读写都是在一个CPU上。

### Redis五大数据类型

#### 1、String

```mysql
set key value #设置key-value
get key  #获取key的value
exists key  #key是否存在
append key value #追加字符串，若key不存，相当于set key value
strlen key #获取字符串长度
incr key #当前key的value加1
decr key #当前key的value减一
incrby key 10 #当前key加10
decrby key 10 #当前key减10
getrange key 0 3 #字符串范围 （getrange key 0 -1 获取全部字符串）
setrange key 1 xx #替换指定位置开始的字符串
setex key second value    #（set with expire）设置过期时间
setnx key value   #（set if not with exists ）不存在再设置 （分布式锁中常使用）
mset key1 v1 key2 v2  #批量设置
mget key1 key2 key3  #批量获取
msetnx key1 v1 key2 v2  #不存在再设置(批量 原子性操作  一起成功 一起失败)
getset key value #先获取原值再设置新值
```

#### 2、List

redis中，可以将list用作栈、队列、阻塞队列的数据结构
所有list命令都是以l开头

```mysql
lpush key v1 v2 ...  #将一个值或多个值插入列表的头部(左)
rpush key v1 v2 ...  #将一个值或多个值插入列表的尾部(右)
lrange key start end  #用过区间获取具体的值  （0 -1 区间获取全部值）
lpop key  #移除列表头部第一个值（左）
rpop key  #移除列表尾部第一个值（右）
lindex key index #通过索引获取值
llen key   #获取列表长度
lrem key count value  #移除list集合中指定个数的value  精确匹配
ltrim key start stop   #通过下标截取指定长度，list已经改变，只剩下截取后的元素
rpoplpush key otherkey  #移除列表中最后一个元素，并将它插入另一个列表头部
lset key index value  #将列表中指定下标的值替换为另外一个值，更新操作 （如果列表或索引不存在  会报错）
linsert key before v1 v2  #在v1前插入v2
linsert key after v1 v2  #在v1后插入v2
```

小结：

- 他实际上是一个链表，before after left right 都可以插入值

- 如果key不存在，创建新的链表

- 如果key存在，新增内容

- 如果移除了所有值，空链表，也代表不存在

- 在两边插入或改动值，效率最高！中间元素，相对来说效率会低一点！

- 消息排队 消息队列（Lpush Rpop） ，栈（Lpush Lpop）

  #### 3、Set

  set中的值不能重复

  ```mysql
  sadd key value  #添加元素
  smembers key   #查看指定set中所用元素
  sismember key value  #判断某一个值在指定set中是否存在
  scard key  #获取set中的内容元素个数
  srem key value   #移除set中指定元素
  srandmember key count  #随机选出指定个数的成员
  spop key  #随机移除元素
  smove oldkey  newkey member  #将一个指定的值，从一个set移动到另一个set
  sdiff key...   #获取多个set差集
  sinter key...  #获取多个set交集 （共同好友、共同关注）
  sunion key...  #获取多个set并集
  ```

  应用场景：微博，将用户所有关注放入一个set，粉丝放入一个set

  -> 共同关注、二度好友、相互关注…..

#### 4、Hash

Map集合 key-(key-value)

```mysql
hset key field value  #存入一个具体键值对
hget key field   #获取一个字段值
hmset key field value field1 value1 ...  #存入多个具体键值对
hmget key field field1 ...  #获取多个字段值
hgetall key    #获取全部数据
hdel key field  #删除hash指定的key字段，对应value也就没有了
hlen key     #获取hash中字段数量
hexists key field   #判断hash中某个字段是否存在
hkeys key    #获取hash中全部key
hvals key    #获取hash中全部value
hincrby key field 1  #hash中指定key的value加1
hdecrby key field 1  #hash中指定key的value减1
hsetnx key field value   #如果hash中指定key不存在则创建，存在则创建失败
```

hash应用场景： 变更的数据，比如用户信息，以及其他经常变动的信息；hash更加适合对象的存储，String更适合字符串的存储。

#### 5、Zset

有序集合，在set的基础上增加了一个排序的值

```mysql
zadd key score value  #添加元素
zrange key 0 1   #通过索引区间返回有序集合指定区间内的成员   （0 -1）返回全部
zrangebyscore key min max   #排序并返回 从小到大  例如：zrangebyscore key1 -inf +inf    （-inf：负无穷   +inf：正无穷 ）
zrevrange key 0 -1     #排序并返回 从大到小
zrem key value   #移除指定元素
zcard key        #获取有序集合中的数量
zcount key start stop   #获取指定区间中的成员数量
```



### Redis三种特殊数据类型

#### 1、geospatial

地理位置 （定位、附近的人、打车距离……）
GEO底层就是Zset 可以用Zset命令操作Geo

```mysql
#geoadd 添加地理位置
规则：两极无法直接加入，通常通过java一次性导入  有效经度：-180到180  有效纬度：-85.05112878到85.05112878
geoadd china:city 121.47 31.23 shanghai
geoadd china:city 106.50 29.53 chongqing  114.05 22.52 shenzhen 120.16 30.24 hangzhou 108.96 34.26 xian
#geopop 获取指定成员的经度和纬度
GEOPOS china:city chongqing beijin
#geodist 查看成员间的的直线距离
GEODIST china:city beijin shanghai km
#georadius 以给定经纬度为中心，找出某一半径内的元素
（附件的人）
GEORADIUS china:city 110 30 1000 km
GEORADIUS china:city 110 30 1000 km withdist withcoord count 2 （withdist 显示直线距离  withcoord 显示经纬度  count  显示几条）
#georadiusbymember 以给定成员为中心，找出某一半径内的元素
georadiusbymember china:city beijing 1000 km withdist withcoord count 2 （withdist 显示直线距离  withcoord 显示经纬度  count  显示几条）
#geohash 返回一个或多个位置元素的geohash表示  将二维的经纬度转换成一维的11位字符串 如果两个字符串越接近，则距离越近。
```

#### 2、hyperloglog

基数 （不重复的元素个数） 可以接受误差 大概有0.81%的错误率
Redis hyperloglog 基数统计算法：
优点： 占用内存固定，存放2^64不同的元素的技术，只需要占用12KB内存
**网页的UV （一个人访问一个网站多次，统计出还是一个人）**
传统的方式：set集合保存用户id，统计set中用户数量。 但是相对消耗更多内存，我们的目的并不是保存用户id，目的只是计数。

```mysql
PFadd key element  #创建一组元素
PFcount key   #统计元素基数
pfmerge key3 key1 key2   #合并两组key1 key2 => key3  并集
```

#### 3、bitmaps

位存储
统计用户信息 活跃 不活跃 登录 未登录 打卡
两个状态的 都可以使用bitmaps
bitmaps位图数据结构，都是操作二进制位来进行记录的，非0即1

```mysql
setbit key offset value  #设置位图
getbit key offset        #获取指定位图的值
bitcount key     #统计数量
###################################################
例如 一周打卡   0为打卡 1打卡
127.0.0.1:6379> SETBIT sign 0 1
(integer) 0
127.0.0.1:6379> SETBIT sign 1 0
(integer) 0
127.0.0.1:6379> SETBIT sign 2 1
(integer) 0
127.0.0.1:6379> SETBIT sign 3 0
(integer) 0
127.0.0.1:6379> SETBIT sign 4 0
(integer) 0
127.0.0.1:6379> SETBIT sign 5 0
(integer) 0
127.0.0.1:6379> SETBIT sign 6 1
(integer) 0
127.0.0.1:6379> GETBIT sign 0
(integer) 1
127.0.0.1:6379> BITCOUNT sign
(integer) 3
```



### Redis事务

**Redis单条命令时保证原子性的，但是事务不保证原子性**
Redis事务的本质：一组命令的集合！一个事务中所有命令都会被序列化，在事务执行的过程中，会按照顺序执行。**一次性、顺序性、排他性**

Redis事务没有隔离级别的概念！
所有命令在事务中，并没有直接执行！只有在发起执行命令的时候才会执行！

Redis的事务：

- 开启事务（multi）

- 命令入队（…）

- 执行事务（exec）\ 放弃事务（discard）

  ```mysql
  127.0.0.1:6379> MULTI
  OK
  127.0.0.1:6379(TX)> set k1 v1
  QUEUED
  127.0.0.1:6379(TX)> set k2 v2
  QUEUED
  127.0.0.1:6379(TX)> get k2
  QUEUED
  127.0.0.1:6379(TX)> set k3 v3
  QUEUED
  127.0.0.1:6379(TX)> EXEC
  1) OK
  2) OK
  3) "v2"
  4) OKmy
  ```

编译型异常（代码有问题，命令有错），事务中所有的命令都不会被执行！

```mysql
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> set k1 v1
QUEUED
127.0.0.1:6379(TX)> get k1
QUEUED
127.0.0.1:6379(TX)> setget k2 v2  #错误命令
(error) ERR unknown command `setget`, with args beginning with: `k2`, `v2`,
127.0.0.1:6379(TX)> set k3 v3
QUEUED
127.0.0.1:6379(TX)> EXEC   #执行事务报错
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get k1   #所有命令都没有被执行
(nil)
127.0.0.1:6379>
```

运行时异常 ，如果事务队列中存在语法性，那么执行命令的时候，其他命令是可以正常执行的，错误命令抛出异常。

```mysql
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> set k1 "gg"
QUEUED
127.0.0.1:6379(TX)> incr k1  #虽然命令报错了，但是事务依旧执行成功了
QUEUED
127.0.0.1:6379(TX)> set k2 v2
QUEUED
127.0.0.1:6379(TX)> get k2
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) (error) ERR value is not an integer or out of range
3) OK
4) "v2"
127.0.0.1:6379>
```

### Redis实现乐观锁

监控！Watch/unwatch(解锁，如果事务执行失败，先解锁，然后再次手动去监视)

- 悲观锁：
  - 很悲观，什么时候都会出问题，无论做什么都会加锁！
- 乐观锁
  - 很乐观，认为什么时候都不会出问题，不会加锁！更新数据的时候判断，在此期间是否有人修改过这个数据。
  - 获取version
  - 更新时比较version

```mysql
127.0.0.1:6379> set k1 100
OK
127.0.0.1:6379> set k2 0
OK
127.0.0.1:6379> WATCH k1  #监控
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> DECRBY k1 20
QUEUED
127.0.0.1:6379(TX)> INCRBY k2 20
QUEUED
127.0.0.1:6379(TX)> EXEC  #执行之前，另外一个线程修改了监控值，导致事务执行失败
(nil)
127.0.0.1:6379>
####################### 在上一个线程EXEC之前 执行以下命令 #########
127.0.0.1:6379> set k1 1000
OK
```



### Jedis

什么是Jedis： Redis官方推荐的java连接开发工具！
1、导入jar包

```xml
<!--导入jedis的包-->
<dependencies>
    <!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>3.2.0</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.62</version>
    </dependency>
</dependencies>
```

2、连接数据库
3、操作命令
4、断开连接

```java
package com.huang;
import redis.clients.jedis.Jedis;

/**
 * @author huangjiale
 */
public class TestPing {
    public static void main(String[] args) {
        //1.new一个jedis对象
        //47.97.43.144
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        //2.jedis所有的命令就是之前学的命令!
        System.out.println(jedis.ping());
    }
}
```



### Springboot整合

在springboot2.X之后，jedis被替换为了lettuce
jedis采用直连，多个线程操作不安全，如果想避免安全问题，使用jedis pool连接池！ 更像BIO模式

lettuce：采用netty，实例可以在多个线程**享，不存在线程不安全的问题，可以减少线程数量。 更像NIO模式
1、导入依赖

```xml
spring-boot-starter-data-redis
```

2、配置连接



### 自定义RedisTemplate

```java
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
/**
 * Redis对json序列化处理
 */
@Configuration
public class RedisConfig_JacksonSerializer
{
    @Bean
    public RedisTemplate <String, Object> getRedisTemplate(RedisConnectionFactory connectionFactory)
    {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(connectionFactory);
        // 序列化配置
        // 使用Jackson2JsonRedisSerialize替换默认序列化方式
        Jackson2JsonRedisSerializer<?> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        //String的序列化
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        //启用默认的类型
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        //序列化类，对象映射设置
        jackson2JsonRedisSerializer.setObjectMapper(om);
        //key使用String序列化
        redisTemplate.setKeySerializer(stringRedisSerializer);
        //value使用jackson序列化
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        //hash的key使用String序列化
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        //hash的value使用jackson序列化
        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
}
```

*RedisUtils*
[RedisUtils-RedisTemplate](https://www.cnblogs.com/zhzhlong/p/11434284.html)

[Redis-StringRedisTemplate](https://www.cnblogs.com/zeng1994/p/03303c805731afc9aa9c60dbbd32a323.html)



### Redis.conf 详解

单位 units

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2021/11/12/kuangstudy301c25a3-90d8-4696-ae2e-f114540a7fe3.png)



包含 includes

![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2021/11/12/kuangstudyaec2405a-6eea-4192-a5a6-917da62c4b1f.png)



网络 network

```xml
bind 127.0.0.1 #绑定访问ip
protected-mode yes #保护模式 yes开启 no关闭
port 6379   #端口
```

