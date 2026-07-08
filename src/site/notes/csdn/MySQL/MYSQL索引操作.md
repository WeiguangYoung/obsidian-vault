---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL索引操作/","title":"MYSQL索引操作","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL索引操作","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493589"}}
---



##### 1. 概述

- **定义**

索引是对数据库表中一列或多列的值进行排序的一种结构，使用索引可快速访问数据库表中的特定信息。


**优缺点**

优点 ： 加快数据检索速度,提高查找效率


缺点 ：占用数据库物理存储空间，当对表中数据更新时,索引需要动态维护,降低数据写入效率


注意 ：


- 通常我们只在经常进行查询操作的字段上创建索引
- 对于数据量很少的表或者经常进行写操作而不是查询操作的表不适合创建索引


##### 2. 索引分类

- 普通(MUL)


普通索引 ：字段值无约束,KEY标志为 MUL


- 唯一索引(UNI)


唯一索引(unique) ：字段值不允许重复,但可为 NULL,KEY标志为 UNI


- 主键索引（PRI）


一个表中只能有一个主键字段, 主键字段不允许重复,且不能为NULL，KEY标志为PRI。通常设置记录编号字段id,能唯一锁定一条记录


##### 3. 索引创建

- 创建表时直接创建索引

```mysql
create table 表名(
字段名 数据类型，
字段名 数据类型，
index 索引名(字段名),
index 索引名(字段名),
unique 索引名(字段名)
);

```
- 在已有表中创建索引：

```mysql
create [unique] index 索引名 on 表名(字段名);

```
```sql
e.g.
create unique index name_index on class(name);

```
- 主键索引添加

```sql
alter table 表名 add primary key(id);

```
- 查看索引

```mysql
1、desc 表名;  --> KEY标志为：MUL 、UNI。
2、show index from 表名;

```
- 删除索引

```mysql
drop index 索引名 on 表名;
alter table 表名 drop primary key;  # 删除主键

```
- 扩展： 借助性能查看选项去查看索引性能

```sql
set  profiling = 1； 打开功能 （项目上线一般不打开）

show profiles  查看语句执行信息

```