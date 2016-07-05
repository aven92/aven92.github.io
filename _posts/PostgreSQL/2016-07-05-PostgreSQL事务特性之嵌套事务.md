---
layout: post
title: PostgreSQL事务特性之嵌套事务
categories: PostgreSQL
---

<!--more-->

嵌套事务的实现是基于SAVEPOINT、ROLLBACK TO SAVEPOINT和RELEASE SAVEPOINT的，也就是设置一个保存点，可以回滚到保存点和释放保存点。

测试表的初始状态如下：

    postgres=# \d test 
         Table "public.test"
     Column |  Type   | Modifiers 
    --------+---------+-----------
     id     | integer | 
     name   | text    | 

    postgres=# select * from test ;
     id | name 
    ----+------
    (0 rows)

开始测试

    postgres=# begin ;
    BEGIN
    postgres=# insert into test values (1, 'a');
    INSERT 0 1
    postgres=# savepoint insert_a;
    SAVEPOINT
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
    (1 row)

    postgres=# insert into test values (2, 'b');
    INSERT 0 1
    postgres=# savepoint insert_b;
    SAVEPOINT
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
      2 | b
    (2 rows)

    postgres=# insert into test values (3, 'c');
    INSERT 0 1
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
      2 | b
      3 | c
    (3 rows)


现在定义了两个SAVEPOINT，并且插入了3条数据，现在测试ROLLBACK TO SAVEPOINT

    postgres=# rollback to insert_b;
    ROLLBACK
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
      2 | b
    (2 rows)

    postgres=# rollback to insert_a;
    ROLLBACK
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
    (1 row)


可见回滚到前面定义的保存点成功了。

如果回滚到前面的保存点，后面的更改就丢失了，包括保存点，比如回滚到insert_a,那么在insert_a之后的数据就没有了，insert_b这个保存点也不存在了。

    postgres=# rollback to insert_a;
    ROLLBACK
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
    (1 row)

    postgres=# rollback to insert_b;
    ERROR:  no such savepoint
    
测试RELEASE SAVEPOINT

    postgres=# select * from test ;
     id | name 
    ----+------
    (0 rows)

    postgres=# begin ;             
    BEGIN
    postgres=# insert into test values (1, 'a');
    INSERT 0 1
    postgres=# savepoint insert_a;              
    SAVEPOINT
    postgres=# select * from test ; 
     id | name 
    ----+------
      1 | a
    (1 row)

    postgres=# insert into test values (2, 'b');
    INSERT 0 1
    postgres=# savepoint insert_b;              
    SAVEPOINT
    postgres=# select * from test ;             
     id | name 
    ----+------
      1 | a
      2 | b
    (2 rows)

    postgres=# release insert_a;
    RELEASE
    postgres=# select * from test ;
     id | name 
    ----+------
      1 | a
      2 | b
    (2 rows)

    postgres=# rollback to insert_a;
    ERROR:  no such savepoint

保存点被释放后就不能再回滚到该保存点了。
