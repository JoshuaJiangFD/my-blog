---
title: MySQL 命令笔记
layout: post
published: true
tags:
	- MySQL
categories:
	- SQL
---


以下是对工作中常用的SQL命令的总结。

### 1. 查询某个表中所有非空字段名

```sql
SELECT `COLUMN_NAME`
FROM `information_schema`.`COLUMNS`
WHERE
`IS_NULLABLE` = 'No'
AND `TABLE_NAME` = 'feed'
AND `TABLE_SCHEMA` = 'prodb_mgmt
```

### 2. 查看一个表的所有字段

[how-to-get-the-sizes-of-the-tables-of-a-mysql-database]( http://stackoverflow.com/questions/9620198/how-to-get-the-sizes-of-the-tables-of-a-mysql-database )

```sql
SELECT
    table_name AS `Table`,
    round(((data_length + index_length) / 1024 / 1024), 2) `Size in MB`
FROM information_schema.TABLES
WHERE table_schema = "$DB_NAME"
    AND table_name = "$TABLE_NAME";
```

### 3. 显示表的建表语句

[show-create-table]( http://dev.mysql.com/doc/refman/5.7/en/show-create-table.html )

```sql
mysql> show create table serverStatusInfo \G;
*************************** 1. row ***************************
       Table: serverStatusInfo
Create Table: CREATE TABLE `serverStatusInfo` (
  `date` date NOT NULL,
  `server` varchar(45) NOT NULL,
  `requestsActiveMax` int(10) unsigned default '0',
  `requestTimeMax` int(10) unsigned default '0',
  `requestTimeMean` float default '0',
  `requestTimeStdDev` float default '0',
  PRIMARY KEY  (`date`,`server`)
) ENGINE=InnoDB DEFAULT CHARSET=gbk
row in set (0.00 sec)
```

### 4. 删除表中所有数据

[Delete Doc](http://dev.mysql.com/doc/refman/5.0/en/delete.html)
[truncate-table doc](http://dev.mysql.com/doc/refman/5.0/en/truncate-table.html)

```sql
delete from tableName;
# Delete: will delete all rows from your table. Next insert will take next auto increment id.
 
truncate tableName;
# Truncate: will also delete the rows from your table but it will start from new row with 1.
```

### 5. LIKE查询中嵌套变量，对字段取子串并强制装换

1. 连接字符串 

	*[CONCAT(str1,str2,...)](http://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_concat)*
2. 强制装换 

	*[CAST(expr AS type)](http://dev.mysql.com/doc/refman/5.7/en/cast-functions.html)*
3. 字符串截断 

	*[SUBSTRING(str,pos), SUBSTRING(str FROM pos), SUBSTRING(str,pos,len), SUBSTRING(str FROM pos FOR len)](http://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring)*
4. 字符串截断从开头到delimer的第count位置为止 

	*[ SUBSTRING_INDEX(str,delim,count)](http://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring-index)*

```sql
 select CAST(SUBSTRING_INDEX(SUBSTRING(feed_url,110),".",1) AS UNSIGNED INTEGER ) as casted, feed_url,id,user_id  from feed where feed_name  like "%自动无线导航%" and add_time> "2016-04-13" and feed_name not like CONCAT('%',user_id,'%');
```

### 6. 对子查询做Join

这个例子是在feed表上对同一个user_id出现了两次feed的所有情况，删除feedid较大的feed。

1.  第一步是子查询查出所有满足条件的user_id,子查询的结果别名为userids
2.  第二步是对userids表做join，查出所有这种情况的feedids，同时在这种情况下再对结果按照user_id做分组，对每一组取MAX(feedid);
3.  第三步是删除得到的feed ids，注意mysql不准许更新的表同时出现在where条件里，因此这里需要再包装一层，建一个临时表，这样就可以删除了。

```sql
# 操作feed表，每个feed有一个id和user_id,找出同一个user下创建了两个feed的user，并删除id较大的feed
update feed 
set is_deleted = 0 
where id in ( select * from (
		select MAX(feed.id) 
      from feed, (select user_id as uid from feed group by user_id having count(*) > 2) as  e )
      where feed.user_id = e.uid 
      group by feed.user_id
	as temp);
```

### 7. 查询表中数据和索引的大小

```sql
# 方法1:切换到 information_schema 数据库
mysql> use information_schema;
# 查询当前数据库中所有的表的数据和
mysql> select concat(round(data_length/1024/1024,2),'MB') as data_length_MB,concat(round(index_length/1024/1024,2),'MB') as index_length_MB, table_name from tables;
# 方法2： show table status命令, 查询prodb_mgmt数据库的所有表情况
mysql> show table status from prodb_mgmt;
```

### 8. 导出查询到csv文件

mysql指定host,port,password,user, databasename, -e选项内填写具体的sql语句，多条语句使用`;`分隔。最后使用重定向符号`>`输出到外部csv文件。

```sql
mysql -hdbbk-prodbpicweb.xdb.all.serv -P6003 -ulufei -pNu99Z2MB6h -Dprodb_mgmt -e 'set character_set_client=utf8;set character_set_results=utf8;select id from feed where feed_name like "%【闪投抓取】bianlian%"' > zhidong-1.csv
```
