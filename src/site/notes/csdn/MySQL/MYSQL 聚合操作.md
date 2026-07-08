---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL 聚合操作/","title":"MYSQL 聚合操作","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL 聚合操作","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493535"}}
---


聚合操作指的是在数据查找基础上对数据的进一步整理筛选行为，实际上聚合操作也属于数据的查询筛选范围。


##### 1. 聚合函数


方法功能avg(字段名)该字段的平均值max(字段名)该字段的最大值min(字段名)该字段的最小值sum(字段名)该字段所有记录的和count(字段名)统计该字段记录的个数eg1 : 找出表中的最大攻击力的值？


```mysql
select max(attack) from sanguo;

```
eg2 : 表中共有多少个英雄？


```mysql
select count(name) as number from sanguo;

```
eg3 : 蜀国英雄中攻击值大于200的英雄的数量


```mysql
select count(*) from sanguo where attack > 200; 

```

注意： 此时select 后只能写聚合函数，无法查找其他字段，除非该字段值全都一样。


##### 2. 聚合分组

- **group by**

给查询的结果进行分组


e.g.  : 计算每个国家的平均攻击力


```mysql
select country,avg(attack) from sanguo 
group by country;

```
e.g. :  对多个字段创建分组，此时多个字段都相同时为一组


```mysql
select age,sex,count(*) from class1 group by age,sex;

```
e.g. : 所有国家的男英雄中 英雄数量最多的前2名的 国家名称及英雄数量


```mysql
select country,count(id) as number from sanguo 
where gender='M' group by country
order by number DESC
limit 2;

```

注意： 使用分组时select 后的字段为group by分组的字段和聚合函数，不能包含其他内容。group by也可以同时依照多个字段分组，如group by A，B 此时必须A,B两个字段值均相同才算一组。


##### 3. 聚合筛选

- **having语句**

对分组聚合后的结果进行进一步筛选


```mysql
eg1 : 找出平均攻击力大于105的国家的前2名,显示国家名称和平均攻击力

select country,avg(attack) from sanguo 
group by country
having avg(attack)>105
order by avg(attack) DESC
limit 2;

```

注意


- having语句必须与group by联合使用。
- having语句存在弥补了where关键字不能与聚合函数联合使用的不足,where只能操作表中实际存在的字段。


##### 4. 去重语句

- **distinct语句**

不显示字段重复值


```mysql
eg1 : 表中都有哪些国家
  select distinct country from sanguo;
eg2 : 计算一共有多少个国家
  select count(distinct country) from sanguo;

```

注意: distinct和from之间所有字段都相同才会去重