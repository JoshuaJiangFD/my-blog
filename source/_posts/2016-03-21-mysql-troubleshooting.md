---
published: true
title: MySQL排障笔记
layout: post
tags:
	- MySQL
categories:
	- SQL
---

###理解mysql client客户端乱码问题

[10分钟学会理解和解决MySQL乱码问题](http://cenalulu.github.io/mysql/mysql-mojibake/)

[mysql中文乱码的一点理解](http://www.ahlinux.com/mysql/21694.html)

[设置linux shell的编码](http://perlgeek.de/en/article/set-up-a-clean-utf8-environment)


```sql
mysql> show variables like 'character_set%';
+--------------------------+-----------------------------------------------------------+
| Variable_name            | Value                                                     |
+--------------------------+-----------------------------------------------------------+
| character_set_client     | gbk                                                       |
| character_set_connection | gbk                                                       |
| character_set_database   | utf8                                                      |
| character_set_filesystem | binary                                                    |
| character_set_results    | gbk                                                       |
| character_set_server     | utf8                                                      |
| character_set_system     | utf8                                                      |
| character_sets_dir       | /home/mysql/mysql_prodbpicweb_46901/share/mysql/charsets/ |
+--------------------------+-----------------------------------------------------------+
8 rows in set (0.00 sec)

mysql> set character_set_results='utf8'; 
Query OK, 0 rows affected (0.01 sec)

```
