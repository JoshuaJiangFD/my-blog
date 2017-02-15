---
title: Linux Bash 小工具
layout: post
published: true
tags:
	- Bash
categories:
	- Linux
---

### 进程 ###

#### 1. ps with headers ####

```bash
# 利用grep的OR条件
$ ps -ef |egrep "PID|[your-pattern-goes-here]"
# 两次查询，第一次获取header，第二次grep实际的查询
ps -ef | head -1; ps -ef | grep "[your-pattern-goes-here]"
```

### 文件 ###

#### 1. find ####

```shell
# find files with modified time 36hours ago and calcualate total size
$ find . -mmin  +$((60*36)) -type f -name "*"  -exec du -hc  {} + 
```
