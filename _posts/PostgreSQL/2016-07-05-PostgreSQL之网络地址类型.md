---
layout: post
title: PostgreSQL之网络地址类型
categories: PostgreSQL
---

<!--more-->

官方文档：http://www.postgresql.org/docs/9.4/interactive/datatype-net-types.html

## 1. cidr

    postgres=# create table test (id int, name text);
    CREATE TABLE
    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 

    postgres=# alter table test add column ip cidr;
    ALTER TABLE
    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 
     ip     | cidr    | 

    postgres=# insert into test values (1, 'a', '192.168.1.100');
    INSERT 0 1
    postgres=# select * from test ;
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
    (1 row)

    postgres=# insert into test values (2, 'b', '192.168.0.0/16');
    INSERT 0 1
    postgres=# select * from test ;
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.0.0/16
    (2 rows)

    postgres=# insert into test values (3, 'c', '192.168.1.0/24');
    INSERT 0 1
    postgres=# select * from test ;
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.0.0/16
      3 | c    | 192.168.1.0/24
    (3 rows)


查询使用

    postgres=# select * from test where ip = '192.168.1.100';
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
    (1 row)

    postgres=#  select * from test where ip >= '192.168.1.0/24';
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      3 | c    | 192.168.1.0/24
    (2 rows)

    postgres=#  select * from test where ip >= '192.168.0.0/16';
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.0.0/16
      3 | c    | 192.168.1.0/24
    (3 rows)

    postgres=# update test set ip = '192.168.1.101/32' where id = 2;
    UPDATE 1
    postgres=# update test set ip = '192.168.1.102/32' where id = 3; 
    UPDATE 1
    postgres=# select * from test ;
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.1.101/32
      3 | c    | 192.168.1.102/32
    (3 rows)

    postgres=# select * from test where  ip between '192.168.1.100' and '192.168.1.101';
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.1.101/32
    (2 rows)

    postgres=# select * from test where  ip between '192.168.1.100' and '192.168.1.102';
     id | name |        ip        
    ----+------+------------------
      1 | a    | 192.168.1.100/32
      2 | b    | 192.168.1.101/32
      3 | c    | 192.168.1.102/32
    (3 rows)

## 2. inet

将cidr修改为inet

    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 
     ip     | cidr    |
     postgres=# alter table test alter column ip type inet;
    ALTER TABLE
    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 
     ip     | inet    |
    
    postgres=# select * from test ;
     id | name |      ip       
    ----+------+---------------
      1 | a    | 192.168.1.100
      2 | b    | 192.168.1.101
      3 | c    | 192.168.1.102
    (3 rows)

    postgres=# update test set ip = '192.168.0.0/16' where id = 3;
    UPDATE 1
    postgres=# select * from test ;                               
     id | name |       ip       
    ----+------+----------------
      1 | a    | 192.168.1.100
      2 | b    | 192.168.1.101
      3 | c    | 192.168.0.0/16
    (3 rows)

    postgres=# update test set ip = '192.168.1.0/24' where id = 2;   
    UPDATE 1
    postgres=# select * from test ;
     id | name |       ip       
    ----+------+----------------
      1 | a    | 192.168.1.100
      3 | c    | 192.168.0.0/16
      2 | b    | 192.168.1.0/24
    (3 rows)

可见，inet默认32位掩码的ip是不带'/32'的

    postgres=# select * from test where ip >= '192.168.1.100';
     id | name |      ip       
    ----+------+---------------
      1 | a    | 192.168.1.100
    (1 row)

    postgres=# select * from test where ip >= '192.168.1.1';  
     id | name |      ip       
    ----+------+---------------
      1 | a    | 192.168.1.100
    (1 row)

    postgres=# select * from test where ip >= '192.168.1.101';
     id | name | ip 
    ----+------+----
    (0 rows)
    postgres=# select * from test where ip >= '192.168.1.0/32';
     id | name |      ip       
    ----+------+---------------
      1 | a    | 192.168.1.100
    (1 row)
    postgres=# select * from test where ip >= '192.168.1.0/16';
     id | name |       ip       
    ----+------+----------------
      1 | a    | 192.168.1.100
      2 | b    | 192.168.1.0/24
    (2 rows)

    postgres=# select * from test where ip >= '192.168.0.0/16';
     id | name |       ip       
    ----+------+----------------
      1 | a    | 192.168.1.100
      3 | c    | 192.168.0.0/16
      2 | b    | 192.168.1.0/24
    (3 rows)

使用跟cidr差不多

## 3. macaddr

    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 
     ip     | inet    | 

    postgres=# alter table test add column mac macaddr;
    ALTER TABLE
    postgres=# \d test
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 
     ip     | inet    | 
     mac    | macaddr | 

    postgres=# select * from test ;
     id | name |       ip       | mac 
    ----+------+----------------+-----
      1 | a    | 192.168.1.100  | 
      3 | c    | 192.168.0.0/16 | 
      2 | b    | 192.168.1.0/24 | 
    (3 rows)

    postgres=# update test set mac = '08:00:2b:01:02:03' where id = 1;
    UPDATE 1
    postgres=# select * from test ;
     id | name |       ip       |        mac        
    ----+------+----------------+-------------------
      3 | c    | 192.168.0.0/16 | 
      2 | b    | 192.168.1.0/24 | 
      1 | a    | 192.168.1.100  | 08:00:2b:01:02:03
    (3 rows)
    postgres=# update test set mac = '08:00:2b:01:02:04' where id = 2; 
    UPDATE 1
    postgres=# update test set mac = '08:00:2b:01:02:05' where id = 3; 
    UPDATE 1
    postgres=# select * from test ;
     id | name |       ip       |        mac        
    ----+------+----------------+-------------------
      1 | a    | 192.168.1.100  | 08:00:2b:01:02:03
      2 | b    | 192.168.1.0/24 | 08:00:2b:01:02:04
      3 | c    | 192.168.0.0/16 | 08:00:2b:01:02:05
    (3 rows)

查询使用

    postgres=# select * from test where mac = '08:00:2b:01:02:03';
     id | name |      ip       |        mac        
    ----+------+---------------+-------------------
      1 | a    | 192.168.1.100 | 08:00:2b:01:02:03
    (1 row)

    postgres=# select * from test where mac > '08:00:2b:01:02:03';
     id | name |       ip       |        mac        
    ----+------+----------------+-------------------
      2 | b    | 192.168.1.0/24 | 08:00:2b:01:02:04
      3 | c    | 192.168.0.0/16 | 08:00:2b:01:02:05
    (2 rows)

PostgreSQL默认还不支持iprange，需要安装ip4r的扩展，详见：http://pgfoundry.org/projects/ip4r/
