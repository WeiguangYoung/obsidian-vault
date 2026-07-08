---
{"dg-publish":true,"permalink":"/csdn/MySQL/mysql密码配置和备份/","title":"mysql密码配置和备份","tags":["mysql","数据库"],"dg-note-properties":{"category":"MySQL","title":"mysql密码配置和备份","source":"csdn","created":"2023-06-12","tags":["mysql","数据库"],"url":"https://blog.csdn.net/weixin_45536921/article/details/131163889"}}
---



#### mysql密码配置和备份

密码配置


```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456' PASSWORD EXPIRE NEVER; 
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456'; 
flush privileges;

```
开放访问


```sql
use mysql;
update user set host="%" where user='root';
flush privileges;

```
数据库备份


```sql
mysqldump -h127.0.0.1 -uroot  -p123456  --tables 
  > backup.sql

```
数据库恢复


```sql
mysql -uroot -p --one-database   ON   TO  [IDENTIFIED BY ""]  [WITH GRANT OPTION];

```
docker快速安装

```
docker run --name mysql -v /mysql:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```