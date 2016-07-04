---
layout: post
title: PostgreSQL权限管理之创建可更新表的普通用户
categories: PostgreSQL
---

<!--more-->

## 1.环境

	$ psql -U postgres
	psql (9.4.4)
	Type "help" for help.
	postgres=# select version();
	                                                    version                                                    
	---------------------------------------------------------------------------------------------------------------
	 PostgreSQL 9.4.4 on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 4.4.7 20120313 (Red Hat 4.4.7-11), 64-bit
	(1 row)
	postgres=# \dt
	No relations found.
	postgres=# revoke select on all tables in schema public from public;
	REVOKE

我们都知道，超级用户的权限太大了，为了数据库的安全，对于非管理员账号，需要创建普通用户。删除public下所有非超户的select权限。

## 2.语法

	$ psql -U postgres
	psql (9.4.4)
	Type "help" for help.

	postgres=# \h create role 
	Command:     CREATE ROLE
	Description: define a new database role
	Syntax:
	CREATE ROLE name [ [ WITH ] option [ ... ] ]

	where option can be:

	      SUPERUSER | NOSUPERUSER
	    | CREATEDB | NOCREATEDB
	    | CREATEROLE | NOCREATEROLE
	    | CREATEUSER | NOCREATEUSER
	    | INHERIT | NOINHERIT
	    | LOGIN | NOLOGIN
	    | REPLICATION | NOREPLICATION
	    | CONNECTION LIMIT connlimit
	    | [ ENCRYPTED | UNENCRYPTED ] PASSWORD 'password'
	    | VALID UNTIL 'timestamp'
	    | IN ROLE role_name [, ...]
	    | IN GROUP role_name [, ...]
	    | ROLE role_name [, ...]
	    | ADMIN role_name [, ...]
	    | USER role_name [, ...]
	    | SYSID uid

## 3.创建只读用户
### (1) 先创建表t1

	postgres=# create table t1 ( id serial, name varchar(64) );
	CREATE TABLE
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | t1   | table | postgres
	(1 row)

### (2) 创建用户u1

	postgres=# create role u1 with login password '123456';
	CREATE ROLE

login是赋予登录权限，否则是不能登录的

### (3) 赋予u1对表的只读权限

因为创建的普通用户默认是没有任何权限的

	postgres=# \c - u1
	You are now connected to database "postgres" as user "u1".
	postgres=> select * from t1;
	ERROR:  permission denied for relation t1
	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# grant select on all tables in schema public to u1;
	GRANT
	postgres=# \c - u1
	You are now connected to database "postgres" as user "u1".
	postgres=> select * from t1;
	 id | name 
	----+------
	(0 rows)

### (4) 创建表t2

	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# create table t2 ( id serial, name varchar(64) );
	CREATE TABLE
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | t1   | table | postgres
	 public | t2   | table | postgres
	(2 rows)

### (5) 验证u1的权限

	postgres=# \c - u1
	You are now connected to database "postgres" as user "u1".
	postgres=> select * from t1;
	 id | name 
	----+------
	(0 rows)

	postgres=> select * from t2;
	ERROR:  permission denied for relation t2

可见u1是有t1表的读权限，但没有t2表的读权限，这样是不是意味着每次新建表就要赋一次权限？

### (6) 解决办法

	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# alter default privileges in schema public grant select on tables to u1;
	ALTER DEFAULT PRIVILEGES
	postgres=# create table t3 ( id serial, name varchar(64) );
	CREATE TABLE
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | t1   | table | postgres
	 public | t2   | table | postgres
	 public | t3   | table | postgres
	(3 rows)

	postgres=# \c - u1
	You are now connected to database "postgres" as user "u1".
	postgres=> select * from t3;
	 id | name 
	----+------
	(0 rows)
	postgres=> select * from t2;
	ERROR:  permission denied for relation t2
	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# grant select on all tables in schema public to u1;
	GRANT
	postgres=# \c - u1
	You are now connected to database "postgres" as user "u1".
	postgres=> select * from t2;
	 id | name 
	----+------
	(0 rows)

grant是赋予用户schema下当前表的权限，alter default privileges是赋予用户schema下表的默认权限，这样以后新建表就不用再赋权限了。当我们创建只读账号的时候，需要执行grant和alter default privileges。

## 4.创建可更新用户
### (1) 创建u2用户

	postgres=# create role u2 with login password '123456';
	CREATE ROLE

### (2) 赋予更新权限

	postgres=# alter default privileges in schema public grant select,insert,update,delete on tables to u2;
	ALTER DEFAULT PRIVILEGES

### (3) 创建表t4

	postgres=# create table t4 ( id serial, name varchar(64) );
	CREATE TABLE
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | t1   | table | postgres
	 public | t2   | table | postgres
	 public | t3   | table | postgres
	 public | t4   | table | postgres
	(4 rows)

### (4) 查看权限

	postgres=# \c - u2
	You are now connected to database "postgres" as user "u2".
	postgres=> insert into t4 values ( 1, 'aa' );
	INSERT 0 1
	postgres=> select * from t4;
	 id | name 
	----+------
	  1 | aa
	(1 row)

	postgres=> update t4 set name = 'bb' where id = 1;
	UPDATE 1
	postgres=> select * from t4;
	 id | name 
	----+------
	  1 | bb
	(1 row)

	postgres=> delete from t4 where id = 1;
	DELETE 1
	postgres=> select * from t4;
	 id | name 
	----+------
	(0 rows)

可以正常增删改查

### (5) 序列的权限与解决办法

在insert的时候，指定列插入，主键id是serial类型会默认走sequence的下一个值，但前面只赋予了表的权限，所以会出现下面的问题：

	postgres=> insert into t4 ( name ) values ( 'aa' );
	ERROR:  permission denied for sequence t4_id_seq

解决方法就是再赋一次sequence的值就行了

	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# alter default privileges in schema public grant usage on sequences to u2;
	ALTER DEFAULT PRIVILEGES
	postgres=# create table t5 ( id serial, name varchar(64) );
	CREATE TABLE
	postgres=# \c - u2
	You are now connected to database "postgres" as user "u2".
	postgres=> insert into t5 ( name ) values ( 'cc' );
	INSERT 0 1
	postgres=> select * from t5;
	 id | name 
	----+------
	  1 | cc
	(1 row)

## 5.删除用户

	postgres=> \c - postgres
	You are now connected to database "postgres" as user "postgres".
	postgres=# drop role u2;
	ERROR:  role "u2" cannot be dropped because some objects depend on it
	DETAIL:  privileges for table t5
	privileges for sequence t5_id_seq
	privileges for default privileges on new sequences belonging to role postgres in schema public
	privileges for table t4
	privileges for default privileges on new relations belonging to role postgres in schema public

当我们删除用户的时候，会提示有权限依赖，所以我们要删除这些权限

	postgres=# alter default privileges in schema public revoke usage on sequences from u2;
	ALTER DEFAULT PRIVILEGES
	postgres=# alter default privileges in schema public revoke select,insert,delete,update on tables from u2;
	ALTER DEFAULT PRIVILEGES
	postgres=# revoke select,insert,delete,update on all tables in schema public from u2;
	REVOKE
	postgres=# revoke usage on all sequences in schema public from u2;
	REVOKE
	postgres=# drop role u2;
	DROP ROLE

