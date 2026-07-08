---
{"dg-publish":true,"permalink":"/csdn/MySQL/MYSQL函数和存储过程/","title":"MYSQL函数和存储过程","tags":["mysql","sql","数据库"],"dg-note-properties":{"category":"MySQL","title":"MYSQL函数和存储过程","source":"csdn","created":"2021-07-05","tags":["mysql","sql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/118493794"}}
---



存储过程和函数是事先经过编译并存储在数据库中的一段sql语句集合，调用存储过程和函数可以简化应用开发工作，提高数据处理的效率。


##### 1. 函数创建

```sql
delimiter 自定义符号　　

create function 函数名(形参列表) returns 返回类型　　-- 注意是retruns
begin
　　函数体   -- 若干sql语句，但是不要直接写查询
　　return val;
end  自定义符号

delimiter ;

释义：
delimiter 自定义符号 是为了在函数内些语句方便，制定除了;之外的符号作为函数书写结束标志,一般用$或者//
形参列表 ： 形参名 类型   类型为mysql支持类型
返回类型:  函数返回的数据类型,mysql支持类型即可
函数体： 若干sql语句组成
return: 返回指定类型返回值

```
```sql
e.g. 无参数的函数调用
delimiter $
create function st() returns int 
begin 
return (select score from class order by score desc limit 1); 
end $
delimiter ;

select st();

```
```sql
e.g. 含有参数的函数调用
delimiter $
create function queryNameById(uid int) 
returns varchar(20)
begin
return  (select name from class where id=uid);
end $
delimiter ;

select queryNameById(1);

```
设置变量
- 定义用户变量 ： set  @[变量名] = 值；使用时用@[变量名]。
- 定义局部变量 ： 在函数内部设置  declare   [变量名]    [变量类型]; 局部变量可以使用set赋值或者使用into关键字。


##### 2. 存储过程创建

创建存储过程语法与创建函数基本相同，但是没有返回值。


```sql
delimiter 自定义符号　

create procedure 存储过程名(形参列表)
begin
　	存储过程　　　　-- sql语句构成存储过程语句集
end  自定义符号

delimiter ;

释义：
delimiter 自定义符号 是为了在函数内些语句方便，制定除了;之外的符号作为函数书写结束标志
形参列表 ：[ IN | OUT | INOUT ] 形参名 类型
          in 输入，out  输出，inout 可以输入也可以输出
存储过程： 若干sql语句组成，如果只有一条语句也可以不写delimiter和begin,end


```
```sql
e.g. 存储过程创建和调用
delimiter $
create procedure st() 
begin 
    select name,age from class; 
    select name,score from class order by score desc; 
end $
delimiter ;

call st();

```
存储过程三个参数的区别
- IN 类型参数可以接收变量也可以接收常量，传入的参数在存储过程内部使用即可，但是在存储过程内部的修改无法传递到外部。
- OUT 类型参数只能接收一个变量，接收的变量不能够在存储过程内部使用（内部为NULL），但是可以在存储过程内对这个变量进行修改。因为定义的变量是全局的，所以外部可以获取这个修改后的值。
- INOUT类型参数同样只能接收一个变量，但是这个变量可以在存储过程内部使用。在存储过程内部的修改也会传递到外部。


```sql
e.g. : 分别将参数类型改为IN OUT INOUT 看一下结果区别
delimiter $
create procedure p_out ( OUT num int )
begin
    select num;
    set num=100;
    select num;
end $

delimiter ;

set @num=10;
call p_out(@num)

```

##### 3. 存储过程和存储函数操作

- 调用存储过程

语法：


```
call 存储过程名字（[存储过程的参数[,……]])

```
- 调用存储函数

语法：


```
select 存储函数名字（[函数的参数[,……]])

```
- 使用show create语句查看存储过程和函数的定义

语法：


```
show create  {procedure|function}  存储过程或存储函数的名称

```
- 查看所有函数或者存储过程

```sql
select name,type from mysql.proc where db='stu';

```
- 删除存储过程或存储函数

语法：


```
DROP {PROCEDURE | FUNCTION} [IF EXISTS] sp_name

```

##### 4. 函数和存储过程区别

- 函数有且只有一个返回值，而存储过程不能有返回值。
- 函数只能有普通参数，而存储过程可以有in,out,inout多个类型参数。
- 存储过程中的语句功能更丰富，实现更复杂的业务逻辑，可以理解为一个按照预定步骤调用的执行过程，而函数中不能展示查询结果集语句，只是完成查询的工作后返回一个结果，功能针对性比较强。
- 存储过程一般是作为一个独立的部分来执行(call调用)。而函数可以作为查询语句的一个部分来调用。