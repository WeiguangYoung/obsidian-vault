---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL数据库基本命令/","title":"MYSQL数据库基本命令","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL数据库基本命令","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118492949"}}
---



## SQL基本操作


#### 1.库操作

- 查看已有库


show databases;


- 创建库


create database 库名 [character set utf8];


```sql
e.g. 创建stu数据库，编码为utf8
create database stu character set utf8;
create database stu charset=utf8;

```

注意：库名的命名


- 数字、字母、下划线,但不能使用纯数字
- 库名区分字母大小写
- 不要使用特殊字符和mysql关键字


- 切换库


use 库名;


```sql
e.g. 使用stu数据库
use stu;

```
- 查看当前所在库


select database();


- 删除库


drop database 库名;


```sql
e.g. 删除test数据库
drop database test;

```

#### 2. 表操作


##### 2.1 基础数据类型

数字类型：
- 整数类型：INT，SMALLINT，TINYINT，MEDIUMINT，BIGINT
- 浮点类型：FLOAT，DOUBLE，DECIMAL
- 比特值类型：BIT


![](/img/user/csdn/assets/a12a87488183.png)


注意：


- 对于准确性要求比较高的东西，比如money，用decimal类型减少存储误差。声明语法是DECIMAL(M,D)。M是数字的最大数字位数，D是小数点右侧数字的位数。比如 DECIMAL(6,2)最多存6位数字，小数点后占2位,取值范围-9999.99到9999.99。
- 比特值类型指0，1值表达2种情况，如真，假


字符串类型：
- 普通字符串： CHAR，VARCHAR
- 存储文本：TEXT
- 存储二进制数据： BLOB
- 存储选项型数据：ENUM，SET


![](/img/user/csdn/assets/26f4b7f260a2.png)


注意：


- char：定长，即指定存储字节数后，无论实际存储了多少字节数据，最终都占指定的字节大小。默认只能存1字节数据。存取效率高。
- varchar：不定长，效率偏低 ，但是节省空间，实际占用空间根据实际存储数据大小而定。必须要指定存储大小 varchar(50)
- enum用来存储给出的多个值中的一个值,即单选，enum(‘A’,‘B’,‘C’)
- set用来存储给出的多个值中一个或多个值，即多选，set(‘A’,‘B’,‘C’)


##### 2.2 表的基本操作

- 创建表


create table 表名(字段名 数据类型 约束,字段名 数据类型 约束,…字段名 数据类型 约束);


字段约束
- 如果你想设置数字为无符号则加上 UNSIGNED
- 如果你不想字段为 NULL 可以设置字段的属性为 NOT NULL， 在操作数据库时如果输入该字段的数据为NULL ，就会报错。
- DEFAULT 表示设置一个字段的默认值
- COMMENT  增加字段说明
- AUTO_INCREMENT 定义列为自增的属性，一般用于主键，数值会自动加1。
- PRIMARY KEY 关键字用于定义列为主键。主键的值不能重复,且不能为空。


```sql
e.g.  创建班级表
create table class (id int primary key auto_increment,name varchar(32) not null,age tinyint unsigned not null,sex enum('w','m'),score float default 0.0);

e.g. 创建兴趣班表
create table hobby (id int primary key auto_increment,name varchar(32) not null,hobby set('sing','dance','draw'),level char not null,price decimal(6,2),remark text);

```

查看数据表


show  tables；


查看表结构


desc  表名;


查看数据表创建信息


show  create  table  表名；


删除表


drop  table  表名;


#### 3. 表数据基本操作


##### 3.1 插入(insert)

```SQL
insert into 表名 values(值1，值2...),(值1，值2...),...;
insert into 表名 (字段1,...) values (值1，值2...),...;

```
```sql
e.g. 
insert into class values (2,'Baron',10,'m',91),(3,'Jame',9,'m',90);

insert into class (name,age,sex,score) values ('Lucy',17,'w',81);


```

##### 3.2 查询(select)

```SQL
select * from 表名 [where 条件];
select 字段1,字段2 from 表名 [where 条件];

```
```sql
e.g. 
select * from class;
select name,age from class;

```

##### 3.3 where子句

where子句在sql语句中扮演了重要角色，主要通过一定的运算条件进行数据的筛选，在查询，删除，修改中都有使用。


- 算数运算符


![](/img/user/csdn/assets/da122df59fc8.png)


```sql
e.g.
select * from class_1 where age % 2 = 0;

```
- 比较运算符


![](/img/user/csdn/assets/ed3e4abc6fb1.png)


```sql
e.g.
select * from class_1 where age > 8;
select * from class_1 where age between 8 and 10;
select * from class_1 where age in (8,9);

```
- 逻辑运算符


![](/img/user/csdn/assets/410934c425fb.png)


```sql
e.g.
select * from class where sex='m' and age>9;

```

![](/img/user/csdn/assets/9053820f3de7.png)


```
查询练习

1. 查找30多元的图书
２．查找人民教育出版社出版的图书　
３．查找老舍写的，中国文学出版社出版的图书　
４．查找备注不为空的图书
５．查找价格超过60元的图书，只看书名和价格
６．查找鲁迅写的或者茅盾写的图书

```

##### 3.4 更新表记录(update)

```SQL
update 表名 set 字段1=值1,字段2=值2,... where 条件;

注意:update语句后如果不加where条件,所有记录全部更新

```
```sql
e.g.
update class set age=11 where name='Abby';

```

##### 3.5 删除表记录（delete）

```SQL
delete from 表名 where 条件;

注意:delete语句后如果不加where条件,所有记录全部清空

```
```sql
e.g.
delete from class where name='Abby';

```

##### 3.6 表字段的操作(alter)

```SQL
语法 ：alter table 表名 执行动作;

* 添加字段(add)
    alter table 表名 add 字段名 数据类型;
    alter table 表名 add 字段名 数据类型 first;
    alter table 表名 add 字段名 数据类型 after 字段名;
* 删除字段(drop)
    alter table 表名 drop 字段名;
* 修改数据类型(modify)
    alter table 表名 modify 字段名 新数据类型;
* 修改字段名(change)
    alter table 表名 change 旧字段名 新字段名 新数据类型;

```
```sql
e.g. 
alter table hobby add tel char(11) after name;
alter table hobby modify tel char(16);
alter table hobby change tel phone char(16);

```

##### 3.7 时间类型数据

- 日期 ： DATE
- 日期时间： DATETIME，TIMESTAMP
- 时间： TIME
- 年份 ：YEAR


![](/img/user/csdn/assets/3fda6d2de253.png)


时间格式


```sql
date ："YYYY-MM-DD"
time ："HH:MM:SS"
datetime ："YYYY-MM-DD HH:MM:SS"
timestamp ："YYYY-MM-DD HH:MM:SS"

```


```sql
e.g.
create table marathon (id int primary key auto_increment,athlete varchar(32),birthday date,registration_time datetime,performance time);

```

日期时间函数


- now()  返回服务器当前日期时间,格式对应datetime类型


时间操作


时间类型数据可以进行比较和排序等操作，在写时间字符串时尽量按照标准格式书写。


```sql
  select * from marathon where birthday>='2000-01-01';
  select * from marathon where birthday>="2000-07-01" and performance<="2:30:00";

```
```
练习 使用book表
1. 将呐喊的价格修改为45元
2. 增加一个字段出版时间 类型为 date 放在价格后面
3. 修改所有老舍的作品出版时间为 2018-10-1
4. 修改所有中国文学出版社出版的但是不是老舍的作品出版时间为 2020-1-1
5. 修改所有出版时间为Null的图书 出版时间为 2019-10-1
6. 所有鲁迅的图书价格增加5元
7. 删除所有价格超过70元或者不到40元的图书


```