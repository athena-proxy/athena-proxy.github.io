---
title: globalID接入及使用
permalink: /docs/dal-globalid/
---

## DAL globalID使用限制
- 保证全局唯一；
- 不保证完全自增，即不保证后面拿到的一定比前面拿到的大；
- 不保证步长有序，即不保证后面拿到的一定和前面拿到的成规律性；
- 支持配合sharding业务，嵌入sharding信息，形成composekey的玩法，具体参考文档。

## 用户申请
需要的信息：
- globalID的业务名称（可以是需要用在的表名）；
- sharding表表名(如果是sharding表的话)；
- shardingKey字段名称（如果是sharding表的话）；
- 用于存储globalid的字段名；
- globalid是否通过shardingKey生成；
- 使用这个globalid的dalgroup列表；
- globalid的预期范围。

## 常见globalID及使用姿势
### common_seq
用户纯粹的分布式唯一ID，无任何嵌入信息。  
#### globalID结构 
+-----------+------------+  
| 1bit sign | 63bit seed |  
+-----------+------------+  

#### 使用姿势  
采用发SQL的方式拿种子。SQL如下：  
```sql
SELECT next_value FROM dal_dual WHERE seq_name = 'common_seq' AND biz = 'xxx_biz'
```
其中common_seq为规则名，是固定的；biz不同的globalID有不同的名称，值一般为业务的领域名或者表名也可。

### composed_seq
背景：在globalID里面嵌入shardingkey的分片信息。主要用于链路中部分业务方不能直接拿到shardingkey那列的信息，导致无法使用sharding特性的情况。解决方式之一是DAL通过在特定列的生成里嵌入此条记录的shardingkey的方式，使得能拿到globalID列的业务方也可进行sharding逻辑的处理，享受到sharding带来的性能提升。  
  
#### globalID结构  
+-----------+------------+------------+  
| 1bit sign | 53bit seed | 10bit hash |  
+-----------+------------+------------+  
10bit hash用于和DAL的分库分表方案配合,通过下述算法，我们把任意数据映射到【0- 1023】的虚拟槽位上。DAL并没有解决分布式事务的问题，而是把分片后的事务限制在单个槽位上，10bithash会用来校验有sharding事务多条SQL的完整性，要求同一个事务中多条SQL对应的虚拟槽位必须相同，也就是我们一直所说的事务跨片问题，实际上这里的片其实是指槽位，而不是分表后的每个分表，比如table_1, table_2之类的。一个分表可能对应多个槽位，所以即便是落在同一个分表的SQL也可能发生事务跨片问题。

DAL并不对外承诺hash算法不变。所以业务上我们只推荐同一个sharding key相关的事务，依赖不同shardingkey计算出hash相同的事务我们不做任何支持。

#### 使用姿势
采用发SQL的方式拿种子。SQL如下：  
```sql
SELECT next_value FROM dal_dual WHERE seq_name = 'composed_seq' AND biz = 'xxx_biz' AND user_id = '123'
```
规则名字固定为'composed_seq', biz不同的Globalld有不同的名字，值一般为业务领域名嵌入Globalld中的信息，条件中的 user_id 为业务sharding表中的shardingkey，值必须为字符串类型，123此处为举例，可以是其它字符串，后续会有算法将此值转换嵌入Globalld中。嵌入Globalld中的信息可以为多组，以样例为例，在user_id = '123'后可以继续附加其它where条件信息嵌入到Globalld中。下面的getHashCode方法实际上摘抄自String 类的hashcode() 方法，然后将计算出的hashcode取绝对值，然后对1024取模，使任何输入的字符串都能映射到【0-1023】的值域内，而表述【0-1023】的值域只需要10bit二进制即可，这10bit二进制即为前面的10bit hash。

```java
public static long getHashCodeValue(Collection<String> list) {
    long hash = 1;
    for (String id : list) {
        if (checkNumID(id)) {
            hash = Math.abs(hash * Long.parseLong(id));
        } else {
            // getHashCodeValue方法原先的实现为:
            // 当传入N个数字a,b,c..时,返回getHashCode(a*b*c*..)
            // 当传入N个非数字a,b,c...时,返回getHashCode(getHashCode(a)*getHashCode(b))
            // 这样就导致一个问题:如果只传入一个非数字时,getHashCode(getHashCode(a))生成的hash返回只能是getHashCode(0-1023),结果只有437个值
            // 所以把它的行为修改为:如果传入list仅有一个值且为非数字型,那么直接返回其hashcode
            // 这样,这个API在传入一个参数时(无论是否是数字型),都能够和Generator.getHashCode(String)保持行为一致
            if (list.size() == 1) {
                return getHashCode(id);
            }
            hash = Math.abs(hash * getHashCode(id));
        }
    }
    return getHashCode(String.valueOf(hash));
}

public static long getHashCode(String id) {
    if (id == null)
        return -1;

    char[] value = id.toCharArray();
    long h = 0;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
    }
    h = Math.abs(h);
    h = h % 1024;
    return h;
}
```