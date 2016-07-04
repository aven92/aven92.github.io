---
layout: post
title: PostgreSQL表空间的使用
categories: PostgreSQL
---

<!--more-->

## 1. 介绍

表空间的作用就是允许数据库管理员定义一个其他非数据目录的位置存储数据库对象。

使用场景之一是，如果机器新加了ssd，但是又不够整个实例使用，可以将一些重要和使用频率高的表和索引放到ssd上，提高查询效率

## 2. 创建表空间

先创建要保存表空间的目录

    # mkdir -p /export/tablespace1
    # chown -R postgres:postgres /export/tablespace1/


进入数据库

    postgres=# \db
           List of tablespaces
        Name    |  Owner   | Location 
    ------------+----------+----------
     pg_default | postgres | 
     pg_global  | postgres | 
    (2 rows)

数据里面只有两个默认的表空间，还没有其他的表空间，现在创建一个表空间

    postgres=# CREATE TABLESPACE tbspl1 LOCATION '/export/tablespace1';
    CREATE TABLESPACE
    postgres=# \db
                 List of tablespaces
        Name    |  Owner   |      Location       
    ------------+----------+---------------------
     pg_default | postgres | 
     pg_global  | postgres | 
     tbspl1     | postgres | /export/tablespace1
    (3 rows)

## 3. 使用表空间

创建一个表table_a并查看其位置

    postgres=# create table table_a (id int);
    CREATE TABLE
    postgres=# select pg_relation_filepath('table_a');
     pg_relation_filepath 
    ----------------------
     base/13003/17031
    (1 row)

可以看出是在数据目录的base目录下，将其修改到表空间tbspl1

    postgres=# alter table table_a set tablespace tbspl1 ;
    ALTER TABLE
    postgres=# select pg_relation_filepath('table_a');     
                 pg_relation_filepath             
    ----------------------------------------------
     pg_tblspc/17030/PG_9.4_201409291/13003/17037
    (1 row)

可见表转移到了表空间所在的目录了

创建使用表空间的表：

    postgres=# create table table_b (id int) tablespace tbspl1;    
    CREATE TABLE
    postgres=# select pg_relation_filepath('table_b');         
                 pg_relation_filepath             
    ----------------------------------------------
     pg_tblspc/17030/PG_9.4_201409291/13003/17038
    (1 row)

对于默认创建的索引是还是在数据目录下的，跟表没有关系

    postgres=# create index on table_b (id);
    CREATE INDEX
    postgres=# \d table_b
        Table "public.table_b"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
    Indexes:
        "table_b_id_idx" btree (id)
    Tablespace: "tbspl1"
    
    postgres=# select pg_relation_filepath('table_b_id_idx');
     pg_relation_filepath 
    ----------------------
     base/13003/17041
    (1 row)

创建使用表空间的索引

    postgres=# create index on table_a (id) tablespace tbspl1; 
    CREATE INDEX
    postgres=# \d table_a
        Table "public.table_a"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
    Indexes:
        "table_a_id_idx" btree (id), tablespace "tbspl1"
    Tablespace: "tbspl1"
    
    postgres=# select pg_relation_filepath('table_a_id_idx'); 
                 pg_relation_filepath             
    ----------------------------------------------
     pg_tblspc/17030/PG_9.4_201409291/13003/17042
    (1 row)
