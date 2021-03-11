---
title: GlobalId原理
permalink: /docs/globalid/
---

## 背景

Globalld最初的目的是为了配合Athena第一个Sharding客户订单表的Sharding方案而产生的一个配套方案。订单表的Sharding方案必须配合Athena提供的Globalld才能工作。后来，由于Athena到DB之间的连接复用，导致原来产研写入一条记录,然后使用 `SELECT LAST_INSERT_ID()`  SQL获取生成的id，然后再赋予此id业务含义的玩法没办法继续下去，因为 `LAST_INSERT_ID() ` 是一个DB会话相关的函数。产研只能使用GloballD预先获取生成的全局唯一id，然后再写入数据库来绕过上述问题。

## 使用姿势

考虑到使用GloballD的用户大部分都是与数据库配套使用的，为了减少业务不必要的依赖，所以Athena提供的GlobalID是通过业务发送特殊SQL的给Athena,然后Athena在返回的结果中返回生成的GloballD给用户使用的。

## GloballD原理

为满足不同业务场景的需求，Athena开发了多种不同种类的Globalld提供给业务使用。大部分返回的Globalld都是类型的结果，基本原理都是在用户的数据库中创建一张 `dal_sequences` 表，然后在该表中写入记录，存储种子值，每个Athena进程在有请求要获取Globalld时会从对应的dal_sequences表中获取对应记录的种子值作为填充位填充到Globalld的高位中保证Globalld的全局唯一性， 也可以通过控制种子值的范围来控制最终生成的Globalld的范围，在Globalld的低位写入不同的信息用于不同的目的。最后将拼凑好的Globalld返回给Client。

同一个DAL Group可以有多个GloballD，多个Globalld的种子值写在同一个Home库中的dal_sequences表中。Home库对于非Sharding的库就等于业务自己的数据库，Sharding库即便Sharding后，依然有些SQL无法找到合适的库执行，而且业务大概率是Sharding表和非Sharding表混用的，原来未sharding的表会存在home库上。此处所说的库是database概念，不是数据库进程级别，所以Home库可以位于独立的数据库机器，同机器不同数据库进程，同进程不同database，甚至可以与某个Sharding分片库相同。业务在请求同一个Athena集群的不同节点时，这些节点都会去同一个dal sequences表获取最新的种子值，每次获取使用 `select for update` 语法保证每次获取种子只会有一个Athena节点成功来避免Globalld重复。

出于性能的考虑，每个Athena节 点会缓存部分的种子值，只有种子值用尽后才会去dal sequences表中获取一段新的种子值缓存。

为了支持多机房部署，同一个逻辑的Globalld在不同机房会使用不同的种子范围，这些范围是保证分段不重合的。也就是说，对于同一个Globalld的种 子在dal_sequences表中会有多条记录，不同的机房使用不同的种子范围。结合下面的种子样例，实际的种子名是逻辑种子名+ezone后缀作为seq name存入的。不同ezone使用不同的种子。

# 实际dal_sequences数据样例

| seq name  | last_ value | is_ delete |
| --------- | ----------- | ---------- |
| xx_ezone1 | 0           | 0          |
| xx_ezone2 | 1000000     | 0          |

# dal_ sequences表结构

```sql
CREATE TABLE `dal_sequences` (
	id bigint(20) unsigned NOT NULL AUTO_ INCREMENT COMMENT 'Physical main id',
	seq_name varchar(64) NOT NULL DEFAULT '' COMMENT 'Sequence name'
	last value bigint(20) NOT NULL DEFAULT '0' COMMENT 'The last used value of the sequence. The next value must be increased based on this value',
	is_delete tinyint(4) NOT NULL DEFAULT '0' COMMENT 'Common column by DA rules',
	created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	updated_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY ('id'),
	UNIQUE KEY `uk_seq_name` (`seq_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='This table stores seeds of global id'
```

