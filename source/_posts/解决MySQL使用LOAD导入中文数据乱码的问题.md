---
title: 解决MySQL使用LOAD导入中文数据乱码的问题
date: 2018-06-03
categories: MySQL
tags: MySQL
---

假设将文本employee.txt中的数据导入到表EMPLOYEE中

出现乱码的SQL语句

```sql
LOAD DATA LOCAL INFILE "employee.txt" INTO TABLE EMPLOYEE;
```

解决方法，在后面添加character set utf8

```
LOAD DATA LOCAL INFILE "employee.txt" INTO TABLE EMPLOYEE character set utf8;
```
