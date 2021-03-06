---
title: Redis数据结构
description: Redis 提供五种数据类型：string，list，hash，set，zset(sorted set)。
---

# 1. string（字符串）
## 1.1. 原理
Redis 的字符串是动态字符串，是可以修改的字符串，内部结构的实现类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配，内部为当前字符串分配的实际空间 capcity 一般要高于实际字符串长度 len。当字符串长度小于1MB时，扩容都是加倍现有的空间。如果字符串长度超过 1MB，扩容时一次只会多扩 1MB 的空间。需要注意的是字符串最大长度为 512MB。
## 1.2. 用法
> 语法

```
set <key> <value>
```

> 设置键值对

```
set name lyh
```

> 获取值

```
get name
```

## 1.3. 应用
> 存储用户信息

```
set user:name lyh
```


# 2. list（列表）

# 3. hash（哈希）

# 4. set（集合）

# 5. zset（有序集合）
