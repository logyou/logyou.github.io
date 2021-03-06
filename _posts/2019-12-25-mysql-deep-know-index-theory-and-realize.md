---
title: 深入理解MySQL索引原理和实现
description: 索引具体是怎么实现的呢？又是如何起作用的呢？
---

我们在实际开发工作中离不开数据库，当用到数据库时又不得不提索引，而索引在数据库中是不可或缺的。但索引具体是怎么实现的呢？又是如何起作用的呢？这篇文章主要探讨这些相关问题。

# 1. 什么是索引

索引（在 MySQL 中也叫做“键（key）”）是存储引擎用于快速找到记录的一种数据结构。这是索引的基本功能。

要理解 MySQL 中索引是如何工作的，最简单的方法就是去看看一本书的“索引”部分：如果想在一本书中找到某个特定主题，一般会先看书的“索引”,找到对应的页码。

在 MySQL 中，存储引擎用类似的方法使用索引，其先在索引中找到对应值，然后根据匹配的索引记录找到对应的数据行。

索引可以包含一个或多个列的值。如果索引包含多个列，那么列的顺序也十分重要，因为 MySQL 只能高效地使用索引的最左前缀列。创建一个包含两个列的索引，和创建两个只包含一列的索引是大不相同的。

# 2. 索引的优点

索引可以让服务器快速地定位到表的指定位置。但是这并不是索引的唯一作用，到目前为止可以看到，根据创建索引的数据结构不同，索引也有一些其他的附加作用。

最常见的 B-Tree 索引，按照顺序存储数据，所以 MySQL 可以用来做 ORDER BY 和 GROUP BY 操作。因为数据是有序的，所以 B-Tree 也就会将相关的列值都存储在一起。最后，因为索引中存储了实际的列值，所以某些查询只使用索引就能够完成全部查询。据此特性，总结下来索引有如下三个优点：
1. 索引大大减少了服务器需要扫描的数据量。
2. 索引可以帮助服务器避免排序和临时表。
3. 索引可以将随机 I/O 变为顺序 I/O。

# 3. 索引的缺点

索引本身也是表，因此会占用存储空间，一般来说，索引表占用的空间是数据表的1.5倍；索引表的维护和创建需要时间成本，这个成本随着数据量增大而增大；构建索引会降低数据表的修改操作（删除，添加，修改）的效率，因为在修改数据表的同时还需要修改索引表。

# 4. 索引的类型

## 4.1 B+Tree 索引

### 4.1.1 定义

B+Tree 是一种多路平衡查找树，是对 B-Tree 的扩展。

一个 m 阶（即：每个节点最多有 m 个键（key））的 B+Tree 满足以下特点:
- 每一个节点最多有 m 个子节点；
- 每一个非叶子节点（除根节点）最少有 ⌈m/2⌉ 个子节点；
- 所有叶子节点所处的高度一样；
- 所有键（key）都会在叶子节点出现；
- 非叶子节点只存储键（key）和指针，只有叶子节点保存数据；
- 键（key）从左到右递增。

### 4.1.2 代码实现

> 树定义

```java
/**
 * B+Tree
 *
 * @author lyh
 * @date 2019/12/25 0025
 **/
public class Tree<K, V> {
    /**
     * 阶数
     */
    private int order;
    /**
     * 根节点
     */
    private Node<K, V> root;
}
```

> 节点定义

```java
/**
 * 节点
 *
 * @author lyh
 * @date 2019/12/25 0025
 **/
public class Node<K, V> {
    /**
     * 父节点
     */
    private Node<K, V> parent;
    /**
     * 前节点
     */
    private Node<K, V> prev;
    /**
     * 后节点
     */
    private Node<K, V> next;
    /**
     * 条目
     */
    private Entry<K, V>[] entries;
}
```

> 条目定义

```
/**
 * 条目
 *
 * @author lyh
 * @date 2019/12/25 0025
 **/
public class Entry<K, V> {
    /**
     * 键
     */
    private K key;
    /**
     * 值
     */
    private V value;
    /**
     * 子节点
     */
    private Node<K, V> child;
}
```

### 4.1.3 结构图

> 表结构

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL,
  `birthday` date DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> 用户数据，id 为主键

id | username | birthday
---|---|---
1 | liu | 1999-10-01
2 | zhang | 1993-12-23
3 | li | 1995-09-14
4 | zhao | 1992-05-03
5 | wang | 1997-01-26
6 | sun | 1991-11-18
7 | zhou | 1996-07-30
8 | shi | 1998-06-02
9 | xie | 1999-03-15
10 | xiong | 1994-08-27
11 | xia | 1996-04-19
12 | zhu | 1996-12-09

> 主键索引，以主键 id 为索引列

![](https://wiki.many.cloud/images/b-plus-tree.jpg)

图片来源：[https://www.processon.com/view/5e0232efe4b0250e8aee9a07](https://www.processon.com/view/5e0232efe4b0250e8aee9a07)

从图中可以看到，节点中每项的键（key）就是主键 id 的值，只有叶子节点存有实际数据。

## 4.2 Hash 索引

### 4.2.1 定义

哈希索引（hash index）基于哈希表实现，只有精确匹配索引所有列的查询才有效。对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码（hash code），哈希码是一个较小的值，并且不同键值的行计算出来的哈希码也不一样。哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每个数据行的指针。

对于 hash code 相同的，采用链表的方式解决冲突。类似于 hashmap，因为索引的结构是十分紧凑的，所以 hash 索引的查询很快。

### 4.2.2 限制

- 哈希索引只包含哈希值和行指针，而不存储字段值，所以不能使用索引中的值来避免读取行。
- 哈希索引数据并不是按照索引值顺序存储的，所以也就无法用于排序。
- 哈希索引也不支持部分索引列匹配查找，因为哈希索引始终是使用索引列的全部内容来计算哈希值的。
- 哈希索引只支持等值比较查询，包括=、IN()、<>（注意<>和<=>是不同的操作）。也不支持任何范围查询，例如WHERE price>100。
- 访问哈希索引的数据非常快，除非有很多哈希冲突（不同的索引列值却有相同的哈希值）。当出现哈希冲突的时候，存储引擎必须遍历链表中所有的行指针，逐行进行比较，直到找到所有符合条件的行。
- 如果哈希冲突很多的话，一些索引维护操作的代价也会很高。例如，如果在某个选择性很低（哈希冲突很多）的列上建立哈希索引，那么当从表中删除一行时，存储引擎需要遍历对应哈希值的链表中的每一行，找到并删除对应行的引用，冲突越多，代价越大。

## 4.3 FullText 索引

### 4.3.1 定义

全文索引是一种特殊类型的索引，它查找的是文本中的关键词，而不是直接比较索引中的值。全文搜索和其他几类索引的匹配方式完全不一样。它有许多需要注意的细节，如停用词、词干和复数、布尔搜索等。全文索引更类似于搜索引擎做的事情，而不是简单的WHERE条件匹配。

在相同的列上同时创建全文索引和基于值的B-Tree索引不会有冲突，全文索引适用于MATCH AGAINST操作，而不是普通的WHERE条件操作。

## 4.4 R-Tree 索引

### 4.4.1 定义

MyISAM 存储引擎支持空间数据索引（R-Tree） ，可以用于地理数据存储。
空间数据索引会从所有维度来索引数据，可以有效地使用任意维度来进行组合查询。必须使用 GIS 相关的函数来维护数据。

# 5. 聚簇索引和非聚簇索引

数据库表的索引从数据存储方式上可以分为**聚簇索引**和**非聚簇索引**（又叫**二级索引**）两种。InnoDB 的聚簇索引在同一个 B+Tree 中保存了索引列和具体的数据，在聚簇索引中，实际的数据保存在叶子页中，中间的节点页保存指向下一层页面的指针。“聚簇”的意思是数据行被按照一定顺序一个个紧密地排列在一起存储。一个表只能有一个聚簇索引，因为在一个表中数据的存放方式只有一种。

**一般来说，将通过主键作为聚簇索引的索引列，也就是通过主键聚集数据。**

## 5.1 聚簇索引

### 5.1.1 定义

聚簇索引并不是一种单独的索引类型，而是一种数据存储方式。具体的细节依赖于其实现方式，但 InnoDB 的聚簇索引实际上在同一个结构中保存了 B+Tree 索引和数据行。

当表有聚簇索引时，它的数据实际上存放在索引的叶子页（leaf page）中。术语‘聚簇’表示数据行和相邻的键值聚簇的存储在一起。因为无法同时把数据行存放在两个不同的地方，所以在一个表中只能有一个聚簇索引 （不过，覆盖索引可以模拟多个聚簇索引的情况）。

因为存储引擎负责实现索引，因此不是所有的存储引擎都支持聚簇索引。

一些数据库服务器允许选择哪个索引作为聚簇索引。InnoDB 将通过主键聚集数据。

如果没有定义主键，InnoDB 会选择一个唯一的非空索引代替。如果没有这样的索引，InnoDB  会隐式定义一个主键来作为聚簇索引。InnoDB 值聚集在同一个页面中的记录。包含相邻键值的页面可能会相距很远。

### 5.1.2 优点

- 可以吧相关的数据保存在一起。例如，实现电子邮箱时，可以根据用户id来聚集数据这样只需要从磁盘读取少数的数据页就能获取某个用户的全部邮件。如果没有使用聚簇索引，则每封邮件都肯能导致一次io。
- 数据访问更快。聚簇索引将索引和数据保存在同一个B-Tree中，因此从聚簇索引中获取数据通常比非聚簇索引中快。
- 使用覆盖索引扫描的查询可以直接使用页节点中的主键值。

### 5.1.3 缺点

- 聚簇索引最大限度的提高了io密集型应用的性能，但如果数据全部存放在内存中，则访问的顺序就没那么重要了，聚簇索引也就没有什么优势了。
- 插入速度严重依赖插入顺序。按照主键的顺序插入是加载数据到innodb表中速度最快的方式。但如果不是按照主键顺序加载数据，那么加载完成后最好使用OPTIMIZE TABLE 命令来重新组织一下表。
- 更新聚簇索引的代价很高，因为会强制InooDB将每个更新的数据移动到新的位置。
- 基于聚簇索引的表在插入行，或者主键被更新导致需要移动行的时候，可能面临’页分裂（page split）‘的问题。当行的主键值要求必须将这一行插入到某个已满的页中时。存储引擎，存储引擎会将该页分裂成两个页面来容纳该行，这就是一次页分裂操作。页分裂会导致表占用更多的存储空间。
- 聚簇索引可能导致全表扫描变慢，尤其是行比较稀疏，或者由于页分裂导致数据存储不连续的时候。
- 　二级索引（非聚簇索引）可能比想象的要更大，因为在二级索引的子节点包含了最优一个几点可能让人有些疑惑，为什么二级索引需要两次索引查找？答案在于二级索引中保存的“行指针”的实质。要记住，二级索引叶子节点保存的不是只想物理位置的指针，而是行的主键值。
- 这意味着通过二级索引进行查找行，存储引擎需要找到二级索引的子节点获得对应的主键值，然后根据这个值去聚簇索引总超找到对应的行。这里做了重复的工作：两次B-Tree查找，而不是一次。对于InnoDB，自适应哈希索引能够减少这样重复工作。

### 5.1.4 结构图

![](https://wiki.many.cloud/images/b-plus-tree.jpg)

图片来源：[https://www.processon.com/view/5e0232efe4b0250e8aee9a07](https://www.processon.com/view/5e0232efe4b0250e8aee9a07)

从图中可以看到，叶子节点存储的是实际数据。

## 5.2 非聚簇索引

### 5.2.1 定义

非聚簇索引，又叫二级索引。二级索引的叶子节点中保存的不是指向行的物理指针，而是行的**主键值**。当通过二级索引查找行，存储引擎需要在二级索引中找到相应的叶子节点，获得行的主键值，然后**使用主键去聚簇索引中查找数据行**，这需要两次 B+Tree 查找。

### 5.2.2 结构图

> 表结构

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL,
  `birthday` date DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> 用户数据，id 为主键

id | username | birthday
---|---|---
1 | liu | 1999-10-01
2 | zhang | 1993-12-23
3 | li | 1995-09-14
4 | zhao | 1992-05-03
5 | wang | 1997-01-26
6 | sun | 1991-11-18
7 | zhou | 1996-07-30
8 | shi | 1998-06-02
9 | xie | 1999-03-15
10 | xiong | 1994-08-27
11 | xia | 1996-04-19
12 | zhu | 1996-12-09

> 二级索引，以 username 为索引列

给 user 表创建一个名为 idx_username 的普通索引 

```sql
CREATE INDEX idx_username ON user (username); 
```

![](https://wiki.many.cloud/images/mysql-non-clustered-index.jpg)

图片来源：[https://www.processon.com/view/5e0380bce4b0250e8af1e391](https://www.processon.com/view/5e0380bce4b0250e8af1e391)

从图中可以看到，叶子节点存储的是主键 id 的值，先找到 username 对应的主键 id 的值，然后再用主键 id 的值去聚簇索引中找到实际数据。

# 6. 索引的语法

## 6.1 普通索引

这是最基本的索引，它没有任何限制。它有以下几种创建方式：

### 6.1.1 创建索引

```sql
CREATE INDEX index_name ON tbl_name (col_name); 
```

示例：给 user 表的 username 列创建一个名为 idx_username 的普通索引

```sql
CREATE INDEX idx_username ON user (username); 
```

### 6.1.2 修改表结构

```sql
ALTER TABLE tbl_name ADD INDEX index_name (col_name)
```

示例：给 user 表的 username 列添加一个名为 idx_username 的普通索引

```sql
ALTER TABLE user ADD INDEX idx_username (username)
```

### 6.1.3 创建表的时候直接指定

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL,
  `birthday` date DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX idx_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 6.1.4 删除索引

```sql
DROP INDEX index_name ON tbl_name
```

示例：从 user 表删除名为 idx_username 的索引

```sql
DROP INDEX idx_username ON user
```


## 6.2 唯一索引

它与前面的普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。它有以下几种创建方式：

### 6.2.1 创建索引

```sql
CREATE UNIQUE INDEX index_name ON tbl_name (col_name)
```

示例：给 user 表的 username 列创建一个名为 unq_username 的唯一索引

```sql
CREATE UNIQUE INDEX unq_username ON user (username)
```

### 6.2.2 修改表结构

```sql
ALTER TABLE tbl_name ADD UNIQUE index_name (col_name)
```

示例：给 user 表的 username 列添加一个名为 unq_username 的唯一索引

```sql
ALTER TABLE user ADD UNIQUE unq_username (username)
```

### 6.2.3 创建表的时候直接指定

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL,
  `birthday` date DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE unq_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

# 7. 高性能的索引策略
## 7.1 独立的列
## 7.2 前缀索引和索引选择性
## 7.3 多列索引
## 7.4 选择合适的索引列顺序
## 7.5 聚簇索引
## 7.6 覆盖索引
## 7.7 使用索引扫描来做排序
## 7.8 压缩（前缀压缩）索引
## 7.9 冗余和重复索引
## 7.10 未使用的索引
## 7.11 索引和锁