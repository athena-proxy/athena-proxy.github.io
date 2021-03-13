---
title: PHP语言接入DAL
permalink: /docs/dal-php/
---

## PHP api现状

目前存在3种PHP api可用于连提MySQL Server, ext/mysql在PHP5被废弃，不要用此api。ext/mysqli功能最全，但是不支持Client端的prepareStatement.  PDO_MySQL支持的功能稍次，但是支持Cient端prepareStatement。

## PHP api选择

DAL出于连接复用的目的不支持server端prepareStatement,如果坚持使用ext/mysqli，则不能使用prepare api相关的接口。无需考虑prepareStatement带来的性能提升，对于性能问题，DAL提供sharding方案解决数据库性能瓶颈。

DAL在非sharding情况下支持Multiple Statements，某些特殊场景可以使用。

如果改动成本不大建议使用PDO_MySQL。

## 事务问题

DAL透明的支持事务，Client可自由使用。无论是autocommit控制还是由start transaction控制。

## 字符集问题

推荐所有应用都使用utf8编码，如果已经是ut8编码则无需对字符集做改动。DAL到DB的连接默认都是 `SET NAMES utf8mb4`  ， 此种设置能够同时支持utf8编码的字符串和和emoji表情符，比如😁。 DAL会忽略应用的SET字符编码的指令。

## session相关的函数

依然是由于连接复用的原因，session相关的函数不再支持。例如 `LAST_INSERT_ID()`，`FOUND_ROWS()`。

对于 `LAST_INSERT_ID()` 函数的需求，可以通过申请DAL的globalid，提前获得全局唯一键来替代insert后获取数据库自增主键的操作。

对于 `LAST_INSERT_ID()` 函数的需求，也可以通过在事务中执行 `SELECT LAST_INSERT_ID()` 获得正确的结果。如下SQL序列：

```sql
set autocommit = 0
insert into t(a) values( 'test')
select last_insert_id()
commit
```

insert类型替换为update，delete类型的SQL，但是不能是select类型的SQL，事务中第一条select类型的SQL在DAL侧不会实际开启事务，DAL的事务是从DML类型的SQL开始的。

如果不想做过多改造，可以使用 `INSERT INTO xxx;SELECT LAST_INSERT_ID()` 或者 `SELECT xXX FROM xxx;SELECT FOUND_ROWS()` 这种Multiple Statements的方式间接使用。这种方式仅适用于非sharding模式。sharding模式下只能改造为globalid的方式。

## 连接池问题

ext/mysqli和PDO_MySQL 都提供连接池化复用，但是都没有办法控制连接最大存活期(一个连接空闲和繁忙加在一起能够存在的最大时间)， DAL 要求控制连接的最大生命期来实现不间断升级，在这种情况下无论是否使用连接池化都有可能受到DAL升级时连接断开的扰动。是否使用连接池化交由应用自己选择，DAL这边的前后连接都是异步的，能够支持的连接上限远高于MySQL Server本身。

## 分库分表问题

查看DAL sharding有关的文档