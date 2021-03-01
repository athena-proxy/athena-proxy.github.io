---
title: Sharding方案限制条件
permalink: /docs/sharding-limit/
---

免责声明:

文档中说的[不支持]代表我们不承诺这类sq|会造成什么后果， 所以不要使用这类sql.可能目前测试出来的结果是你预期的结果，但是我们不保证是否存在其他的sql产生的结果不符合预期，或者后面dal改造产生的变更也可能导致这类sq行为的变更。使用[不支持]的sql产生的任何生产问题，由使用方自己背锅。

## 业务接入sharding准备工作:

1. 了解业务需求，查询条件where是否符合dal的sharding规范，INSERT/UPDATE/DELETE 等事务操作，条件中必须要有Sharding key.

2. 了解业务目前数据量，对sharding需求的紧急与否.

3. 如果该业务已经接入过DAL ,那么抓线上日志查看SQL Pattern(可选).

4. 需要业务提供未来数据量和峰值QPS.

5. 原表的id字段在sharding后不能保证全局唯一，建议去掉对id字段的依赖。如需全局唯一的序列来赋予业务含义， 请新增一个字段来存储全局唯一id (即不要使用id字段来存globaild) 并使用globalid方案，预先获取到唯一值，然后写入到数据库中。

6. 如果有globalid,那么DAL生成的id不能保证绝对自增
   原因是globlld通过种子(确定唯一性的自增序列)和其它的嵌入信息(比如shardingKey的hash值)合成的， 所以不是自增。

7. 如果有globalid,那么必须加入到dal-globalids yml中.

8. 对于shardingKey, composecKey, mappingkey插入后不可以用update修改shardingKey, composedKey, mappingkey.可以通过先delete后再insert方式代替update.
   如果update更新了shardingkey的值，改变了data row与表的分片规则，比如原值是10，update成了11，按照路由规则就变了，但是这条记录还在10对应的分表里面，当业务按照新的值11去查询这条记录的时候，就会发现查询不到这条记录(因为update之后记录还在10对应的表里面) .

9. 对于mapping方案，需要注意3点:
   a.采用mapping方案的前提是mappingkey和shardingkey是一对一或者多对的对应关系，不支持一对多或多对多的关系。

   b. 如果在insert时无法构建mapping关系(无法获取mappingkey的值)，需要申请新的sharding表存mapping关系，由业务方自行维护。

   c. 针对映射表sharding方案，确认写入映射关系后，不允许再更新mapping key。如果确实有更新需求，则用先delete,后重 新insert的方式代替。

10. 需注意和改造如下SQL:

a.如果有多表事务,那么必须是操作同一个shardingKey,否则不支持

b.不支持在事务中会操作非sharding表

C.只支持部分聚合函数(多分片的情况下，DAL只支持COUNT,SUM,MAX,MIN)

d.不支持多分片的group by

e.不支持多分片的order by, (如有需要，可使用sharding_index=?的功能来部分实现order by+limit)

f. shrding分片总数获得方式: SELECT sharding count FROM dal dual where table_name = 'xxx_order'

g.不支持多分片的offset

h.不支持update/delete的where条件不带shardingkey, mappingkey (shardingkey < ?或shardingkey> ?等条件也不支持)

i.在多维sharding的情况下,不支持delete/update where sharding字段= ?,如不支持delete from xxx_order where user_id = ?,因为无法删除其它维度的数据。

j.不支持JOIN或UNION语句，不支持子查询和用分号拼成的一-条SQL.

k. shardingKey, composedkey 或mappingKey必须出现在"=", "IN"操作中，"not in", "!=","<>", ">" "<" ">=", "<="表达式中的shardingkey, mappingkey, composedKey无效，等同于不存在。1.不同where条件间的逻辑连接，最好只用"AND"连接。换句话说，最好where条件只有where a = ? AND b in (?, ?) AND shardingkey= ?。

m.如果你真的不可避免的在SQL中出现了用"OR"连接的where条件，请自行搜索理解AST(Abstract Syntax Tree), DAL是对SQL的分析是基于AST的。有效的Sharding条件定义为: SQL中存在使用"="或"IN"表述的sharding条件，且存在sharding条件所在 节点到顶层where条件之间经过的AST路径都是AND连接。下面有2个例子，看得明白最好，看不明白就看上一条， 不要在where条件中使用OR逻辑连接:

- 这条SQL:

```sql
# sharding table: xxx_order
# shardingkey: restaurant_id或 user_id
SELECT * FROM xxx_order WHERE xxx_order.restaurant_id = 100 and (user_id = 1500 or abc=123);
```

将按照restaurant jid = 100这个条件精确查询一个分片表。

- 这条SQL:

```sql
# sharding table: xxx_order
# shardingkey: restaurant_id或 user_id
SELECT xxx_order.id AS xxx_order_id FROM xxx_order WHERE xxx_order.restaurant id IN (2000,3000) or x=y
```

将会扫描所有的分片表。

n. sharding之后，不建议按照原先的自增主键进行查询,结果可能不符合预期

o.如果存在多个dalgroup都需要获取globalid时,不允许多个dalgroup去获取相同sharding表的globalid,即同一个globalid只能隶属于单个dalgroup

P.如果SQL中有shardingkey between ? and?, DAL对此语法视而不见。对于select SQL,将会采取扫全片的方式查询

q.只支持select/update/insert/delete/truncate/show语句

r.不支持replace into, on duplicate key update,替换为insert into...  select...form dual where not exists (..);

s. MySQL sharding的语句中值必须使用单引号，不支持双引号

t. where in (mappingkey)当mappingkey数量大于分表个数时，会不再查询mapping表，全直接全部扫业务sharding表，建议in mappingkey数量不要太多，可以分开查询。

u. where in (shardingkey)当shardingkey数量不建议大于sharding count,会很慢。若想提高每条SQL的执行时间，建议进一步减少传入shardingkey的数量。

v.关于分页，单片的limit操作，不受影响，直接发到数据库。

跨片的limit操作，目前提供了一种非严格语义上的imit,具体如下:

比如sq|为limit100, 第一个分片返回了50条记录，第二个分片返回了80条记录，我们会直接取第一个分片的50条和第二个分片的前50条返回给用户，其余的分片不再访问。

11.如果业务需要生成globalid,那么需要在相应环境里初始化种子

12.生产环境db资源准备

13.如果该业务需要老数据那么DAL需要在线上切换sharding之前导入老数据

a.导入数据时需要和业务方确认每张表的唯一一键,以便dal可以多次导入数据而不会出现数据重复

b.也防止切换origin->sharding失败，回退sharding->origin时,不会出现id重复等问题

C.导数据时候建议加一个新的group,可以单独看其metrics,记得zkdal init这个group

14.和业务方确认业务低谷期间,此时业务会处于宕机状态,进行切换操作

## 注意事项

1.确认非sharding库和sharding库的参数配置-致， 目前已知的一-定要check的参数如下:

a. sql _mode

b.字符集

C. explicit_defaults_for_timestamp

2.确认DAL是否开启了SHARDING模式

3.确认业务方是否有batch insert行为，如有需打开开关此功能已不再开放支持。

4.目前DAL不支持值为blob(varbinary)的sharding

5.sharding表，unique index不能保证数据的全局唯一-性

## Q&A

1.在排障时，如何查询对应的sharding. key对应到哪个片?

A:使用select help from dal dual; 查看select shard. _info对应的查询语句。

(注意，这些语句仅做排障使用，不要在代码中使用这些对应的查询语句，如果你写了，请做好背锅准备! )

2.觉得dal层是串行的太慢了，我想查询到对应的片区，然后并发查询，要怎么做?

A:方案一，有shardkey, 则拆了sql, 并发查。

方案二，如果没有shardkey,可参考文档sharding表全表扫描方案，注意可能会触发dal限流。

(注意，不建议查询到对应的片区，然后并发查询(dal后期不支持。)

3.希望提高sharding扫全片性能例切sql in语句中句含大景ShardKey由干dal是串行执行，耗时较长

A:方案一，有shardkey, 则拆了sql,并发查。

方案二，如果没有shardkey,可参考文档sharding表全表扫描方案，注意可能会触发dal限流。

(注意，不建议查询到对应的片区，然后并发查询(dal 后期不支持。)
