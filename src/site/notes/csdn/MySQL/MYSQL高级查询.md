---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL高级查询/","title":"MYSQL高级查询","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL高级查询","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493290"}}
---



模糊查询


LIKE用于在where子句中进行模糊查询，SQL LIKE 子句中使用百分号```
%
```来表示任意0个或多个字符，下划线```
_
```表示任意一个字符。


```sql
SELECT field1, field2,...fieldN 
  FROM table_name
  WHERE field1 LIKE condition1

```
```sql
mysql> select * from class where name like 'A%';

```


as 用法


在sql语句中as用于给字段或者表重命名


```sql
select name as 姓名,age as 年龄 from class;
select * from class as cls where cls.age > 17;

```


排序


ORDER BY 子句来设定你想按哪个字段哪种方式来进行排序，再返回搜索结果。


使用 ORDER BY 子句将查询数据排序后再返回数据：


```sql
SELECT field1, field2,...fieldN from table_name1 where field1
ORDER BY field1 [ASC [DESC]]

```
默认情况ASC表示升序，DESC表示降序


```sql
select * from class where sex='m' order by age desc;

```
复合排序：对多个字段排序，即当第一排序项相同时按照第二排序项排序


```sql
select * from class order by score desc,age;

```


限制


LIMIT 子句用于限制由 SELECT 语句返回的数据数量 或者 UPDATE,DELETE语句的操作数量


带有 LIMIT 子句的 SELECT 语句的基本语法如下：


```sql
SELECT column1, column2, columnN 
FROM table_name
WHERE field
LIMIT [num] [OFFSET num]

```
```sql
查询班级男生第三名
select * from cls where sex='m' order by score desc limit 1 offset 2;

```


联合查询


UNION 操作符用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中。多个 SELECT 	语句会删除重复的数据。


UNION 操作符语法格式：


```sql
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];

```
默认UNION后卫 DISTINCT表示删除结果集中重复的数据。如果使用ALL则返回所有结果集，	包含重复数据。


```sql
select * from class where sex='m' UNION ALL select * from class_1 where age > 9;

```


子查询


定义 ： 当一个语句中包含另一个select 查询语句，则称之为有子查询的语句


子查询使用位置：


- from 之后 ，此时子查询的内容作为一个新的表内容，再进行外层select查询

```sql
select name from (select * from class where sex='m') as s where s.score > 90;

```

注意：  需要将子查询结果集重命名一下，方便where子句中的引用操作


- where子句中，此时select查询到的内容作为外层查询的条件值

```sql
 	select *  from class where age = (select age from class where name='Tom');
 	select * from class where name in (select name from hobby);

```

注意：


- 子句结果作为一个值使用时，返回的结果需要一个明确值，不能是多行或者多列。
- 如果子句结果作为一个集合使用，即where子句中是in操作，则结果可以是一个字段的多个记录。


查询过程


通过之前的学习看到，一个完整的select语句内容是很丰富的。下面看一下select的执行过程：


```
(5)SELECT DISTINCT                      

(1)FROM   JOIN  ON 

(2)WHERE 

(3)GROUP BY 

(4)HAVING 

(6)ORDER BY 

(7)LIMIT 

```
```sql
高级查询练习

在stu下创建数据报表 sanguo

字段：id  name  gender  country  attack  defense

create table sanguo(
id int primary key auto_increment,
name varchar(30),
gender enum('男','女'),
country enum('魏','蜀','吴'),
attack smallint,
defense tinyint
);


insert into sanguo
values (1, '曹操', '男', '魏', 256, 63),
       (2, '张辽', '男', '魏', 328, 69),
       (3, '甄姬', '女', '魏', 168, 34),
       (4, '夏侯渊', '男', '魏', 366, 83),
       (5, '刘备', '男', '蜀', 220, 59),
       (6, '诸葛亮', '男', '蜀', 170, 54),
       (7, '赵云', '男', '蜀', 377, 66),
       (8, '张飞', '男', '蜀', 370, 80),
       (9, '孙尚香', '女', '蜀', 249, 62),
       (10, '大乔', '女', '吴', 190, 44),
       (11, '小乔', '女', '吴', 188, 39),
       (12, '周瑜', '男', '吴', 303, 60),
       (13, '吕蒙', '男', '吴', 330, 71);

查找练习
1. 查找所有蜀国人信息，按照攻击力排名
2. 将赵云攻击力设置为360，防御设置为70
3. 吴国英雄攻击力超过300的改为300，最多改2个
4. 查找攻击力超过200的魏国英雄名字和攻击力并显示为姓名， 攻击力
5. 所有英雄按照攻击力降序排序，如果相同则按照防御生序排序
6. 查找名字为3字的
7. 查找攻击力比魏国最高攻击力的人还要高的蜀国英雄
8. 找到魏国防御力排名2-3名的英雄
9. 查找所有女性角色中攻击力大于180的和男性中攻击力小于250的


```