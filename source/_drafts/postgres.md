---
title: postgres
tags:
---

如何导出PostgreSQL数据库中的数据：

pg_dump -U postgres(用户名)  (-t 表名)  数据库名(缺省时同用户名)  > 路径/文件名.sql

```
$ pg_dump -U postgres -d mydatabase -f dump.sql
```

导入数据时首先创建数据库再用psql导入：

$ psql -d databaename(数据库名) -U username(用户名) -f < 路径/文件名.sql  // sql 文件不在当前路径下


```
$ createdb newdatabase
$ psql -d newdatabase -U postgres -f dump.sql
```
