---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL外键约束和表关联/","title":"MYSQL外键约束和表关联","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL外键约束和表关联","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493666"}}
---



##### 1. 外键约束

- 约束 : 约束是一种限制，它通过对表的行或列的数据做出限制，来确保表的数据的完整性、关联性
- foreign key 功能 : 建立表与表之间的某种约束的关系，由于这种关系的存在，能够让表与表之间的数据，更加的完整，关连性更强，为了具体说明创建如下部门表和人员表。
- 示例

```sql
# 创建部门表
CREATE TABLE dept (id int PRIMARY KEY auto_increment,dname VARCHAR(50) not null);

```
```sql
# 创建人员表
CREATE TABLE person (
  id int PRIMARY KEY AUTO_INCREMENT,
  name varchar(32) NOT NULL,
  age tinyint unsigned,
  salary decimal(8,2),
  dept_id int
) ;

```
上面两个表中每个人员都应该有指定的部门，但是实际上在没有约束的情况下人员是可以没有部门的或者也可以添加一个不存在的部门，这显然是不合理的。


- 主表和从表：若同一个数据库中，B表的外键与A表的主键相对应，则A表为主表，B表为从表。


foreign key 外键的定义语法：


```sql
[CONSTRAINT symbol] FOREIGN KEY（外键字段） 

REFERENCES tbl_name (主表主键)

[ON DELETE {RESTRICT | CASCADE | SET NULL | NO ACTION}]

[ON UPDATE {RESTRICT | CASCADE | SET NULL | NO ACTION}]

```
该语法可以在 CREATE TABLE 和 ALTER TABLE 时使用


```sql
# 创建表时直接简历外键
CREATE TABLE person (
  id int PRIMARY KEY AUTO_INCREMENT,
  name varchar(32) NOT NULL,
  age tinyint unsigned,
  salary decimal(10,2),
  dept_id int ,
  constraint dept_fk foreign key(dept_id) references dept(id));

```


建立表后增加外键


```
alter table person add constraint dept_fk foreign key(dept_id) references dept(id);

```

注意：


- 并不是任何情况表关系都需要建立外键来约束，如果没有类似上面的约束关系时也可以不建立。
- 从表的外键字段数据类型与指定的主表主键应该相同。


通过外键名称解除外键约束


```sql
alter table person drop foreign key dept_fk;

# 查看外键名称
show create table person;

```

注意：删除外键后发现desc查看索引标志还在，其实外键也是一种索引，需要将外键名称的索引删除之后才可以。


级联动作


restrict(默认)  :  on delete restrict  on update restrict
- 当主表删除记录时，如果从表中有相关联记录则不允许主表删除
- 当主表更改主键字段值时，如果从表有相关记录则不允许更改


cascade ：数据级联更新  on delete cascade   on update cascade
- 当主表删除记录或更改被参照字段的值时,从表会级联更新


set null  :   on delete set null    on update set null
- 当主表删除记录时，从表外键字段值变为null
- 当主表更改主键字段值时，从表外键字段值变为null


##### 2. 表关联关系

当我们应对复杂的数据关系的时候，数据表的设计就显得尤为重要，认识数据之间的依赖关系是更加合理创建数据表关联性的前提。一对多和多对多是常见的表数据关系：


- 一对多关系


一张表中有一条记录可以对应另外一张表中的多条记录；但是反过来，另外一张表的一条记录

只能对应第一张表的一条记录，这种关系就是一对多或多对一


举例： 一个人可以拥有多辆汽车，每辆车登记的车主只有一人。


```sql
create table person(
  id varchar(32) primary key,
  name varchar(30),
  age int
);

create table car(
  id varchar(32) primary key,
  brand varchar(30),
  price decimal(10,2),
  pid varchar(32),
  foreign key(pid) references person(id)
);

```
- 多对多关系


一对表中（A）的一条记录能够对应另外一张表（B）中的多条记录；同时B表中的一条记录

也能对应A表中的多条记录


举例：一个运动员可以报多个项目，每个项目也会有多个运动员参加,这时为了表达多对多关系需要单独创建关系表。


```mysql
CREATE TABLE athlete (
  id int primary key AUTO_INCREMENT,
  name varchar(30),
  age tinyint NOT NULL,
  country varchar(30) NOT NULL
);

CREATE TABLE item (
  id int primary key AUTO_INCREMENT,
  rname varchar(30) NOT NULL
);

CREATE TABLE athlete_item (
   id int primary key auto_increment,
   aid int NOT NULL,
   tid int NOT NULL,
   FOREIGN KEY (aid) REFERENCES athlete (id),
   FOREIGN KEY (tid) REFERENCES item (id)
);

```

##### 3. E-R模型图

- **定义**

```mysql
E-R模型(Entry-Relationship)即 实体-关系 数据模型,用于数据库设计
用简单的图(E-R图)反映了现实世界中存在的事物或数据以及他们之间的关系

```
- **实体、属性、关系**

​	实体


```mysql
1、描述客观事物的概念
2、表示方法 ：矩形框
3、示例 ：一个人、一本书、一杯咖啡、一个学生

```
​	属性


```mysql
1、实体具有的某种特性
2、表示方法 ：椭圆形
3、示例
   学生属性 ：学号、姓名、年龄、性别、专业 ... 
   感受属性 ：悲伤、喜悦、刺激、愤怒 ...

```
​	关系


```mysql
1、实体之间的联系
2、一对多关联(1:n)
3、多对多关联(m:n) 

```
- **E-R图的绘制**

矩形框代表实体,菱形框代表关系,椭圆形代表属性


![](/img/user/csdn/assets/2284e3fcdc7a.png)


##### 4. 表关联查询

如果多个表存在一定关联关系，可以多表在一起进行查询操作，其实表的关联整理与外键约束之间并没有必然联系，但是基于外键约束设计的具有关联性的表往往会更多使用关联查询查找数据。


- 简单多表查询

多个表数据可以联合查询，语法格式如下：


```sql
select  字段1,字段2... from 表1,表2... [where 条件]

```
```sql
e.g.
select * from class,hobby where class.name = hobby.name;

```
笛卡尔积现象就是将A表的每一条记录与B表的每一条记录强行拼在一起。所以，如果A表有n条记录，B表有m条记录，笛卡尔积产生的结果就会产生n*m条记录。


```sql
select * from class,hobby;

```
- 内连接

内连接查询只会查找到符合条件的记录，其实结果和表关联查询是一样的,官方更推荐使用内连接查询。


![](/img/user/csdn/assets/7a05b5a5d993.png)


```sql
SELECT 字段列表
    FROM 表1  INNER JOIN  表2
ON 表1.字段 = 表2.字段;

```
```sql
select * from person inner join  dept  on  person.dept_id =dept.id;

```
- 左连接  : 左表全部显示，显示右表中与左表匹配的项


![](/img/user/csdn/assets/c98d28dba09e.png)


```sql
SELECT 字段列表
    FROM 表1  LEFT JOIN  表2
ON 表1.字段 = 表2.字段;

```
```sql
select * from person left join  dept  on  person.dept_id =dept.id;

# 查询每个部门员工人数
select dname,count(name) from dept left join person on dept.id=person.dept_id group by dname;

```
- 右连接 ：右表全部显示，显示左表中与右表匹配的项


![](/img/user/csdn/assets/01d7d08f978a.png)


```sql
SELECT 字段列表
    FROM 表1  RIGHT JOIN  表2
ON 表1.字段 = 表2.字段;

```
```sql
select * from person right join  dept  on  person.dept_id =dept.id;

```

注意：我们尽量使用数据量大的表作为基准表,放在前面。


```sql
综合查询练习

create table class(cid int primary key auto_increment,
                  caption char(4) not null);
                  
create table teacher(tid int primary key auto_increment,
                    tname varchar(32) not null);
                    
create table student(sid int primary key auto_increment,
                    sname varchar(32) not null,
                    gender enum('male','female','others') not null default 'male',
                    class_id int,
                    foreign key(class_id) references class(cid)
                    on update cascade
                    on delete cascade);
                    
create table course(cid int primary key auto_increment,
                   cname varchar(16) not null,
                   teacher_id int,
                   foreign key(teacher_id) references teacher(tid)
                   on update cascade
                   on delete cascade);
                   
create table score(sid int primary key auto_increment,
                  student_id int,
                  course_id int,
                  number int(3) not null,
                  foreign key(student_id) references student(sid)
                   on update cascade
                   on delete cascade,
                   foreign key(course_id) references course(cid)
                   on update cascade
                   on delete cascade);
                   
insert into class(caption) values('三年二班'),('三年三班'),('三年一班');
insert into teacher(tname) values('波多老师'),('苍老师'),('小泽老师');
insert into student(sname,gender,class_id) values('钢蛋','female',1),('铁锤','female',1),('山炮','male',2),('彪哥','male',3);
insert into course(cname,teacher_id) values('生物',1),('体育',1),('物理',2);
insert into score(student_id,course_id,number) values(1,1,60),(1,2,59),(2,2,100),(3,2,78),(4,3,66);

1. 查询每位老师教授的课程数量
2. 查询学生的信息及学生所在班级信息
3. 查询各科成绩最高和最低的分数,形式 : 课程ID  课程名称 最高分  最低分
4. 查询平均成绩大于85分的所有学生学号,姓名和平均成绩
5. 查询课程编号为2且课程成绩在80以上的学生学号和姓名
6. 查询各个课程及相应的选修人数
7. 查询每位学生的姓名，所在班级和各科平均成绩

```