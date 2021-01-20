[TOC]

# 一、redis基础

## 1. redis定义

> 中文官网：http://www.redis.cn/

## 2. redis基本命令

```bash
auth <password>	# 若redis配置密码，则需要认证
select <db_index> # 切换分片
keys * # 获取当前分片全部key
dbsize # 查看当前分片key数量
flushdb # 清除当前分片全部数据
flushall # 清除所有分片的数据 
```



## 3.  redis数据类型

### 基本类型

#### （1）string

> 小技巧：可使用a:1表示id为1的对象
>
> ```bash
> # 基础命令
> set key value
> get key
> del key [key...]
> append key value # 追加value
> mset key value [key value...]
> exists key [key...] # 查看key是否存在
> # 分布式锁相关（重点）
> setnx key value # 如果不存在则set值return 1，否则直接return 0;msetnx同理
> # 过期时间操作
> setex key second value # 设置key过期时间，单位：s
> expire key second # 设置key过期时间，单位：s
> ttl key # 获取key有效时间，单位：s
> # 自增自减操作
> incr key # 自增 value+1
> decr key # 自减 value-1
> incrby key increment # 自增 value+increment
> decrby key increment # 自减 value-increment
> ```

#### （2）hash

> hash的语法与string相似，自增自减，get、set都很像。
>
> 使用场景：很适合存对象结构
>
> ```bash
> hset key field value [filed value...]
> hget key field # 获取hash中的field的val
> hmget key field [field...] # 批量获取hash中的field的val
> hdel key field [field...] # 删除hash内的一个或多个filed
> hexists key field # 判断field是否存在于hash中
> # 取整个hash的操作
> hlen key # 获取hash的field数量
> hgetall key # 从hash中取全部的key和val
> hkeys key # 获取hash的所有key
> hvals key # 获取hash的所有val
> ```

#### （3）list

>使用场景：可以当队列、堆栈、列表等数据结构
>
>```bash
>
>```
>
>

#### （4）set

>
>
>

#### （5）zset

### 高级类型

#### （1）geo

#### （2）loglog

#### （3）bitmap

## 4. 发布/订阅

```bash
publish <channel> <message> # 发布消息
```



## 5. redis配置文件



## 6. redis持久化

### （1）rdb



### （2）aof



## 7. redis主从复制



## 8. redis哨兵模式



## 9. redis 集群模式