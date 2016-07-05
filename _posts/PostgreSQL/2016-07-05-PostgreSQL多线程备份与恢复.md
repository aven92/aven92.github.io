---
layout: post
title: PostgreSQL多线程备份与恢复
categories: PostgreSQL
---

<!--more-->

## 1.环境

CentOS7下8核的cpu，数据库的配置是initdb后默认的：

	$ cat /etc/redhat-release
	CentOS Linux release 7.0.1406 (Core)
	$ uname -a
	Linux zhaopin-2-203 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
	$  cat /proc/cpuinfo  | grep processor | wc -l
	8
	$ psql --version
	psql (PostgreSQL) 9.4.5
	$ psql -U postgres -d test
	psql (9.4.5)
	Type "help" for help.

	test=# \dt+
	                    List of relations
	 Schema | Name  | Type  |  Owner   |  Size  | Description
	--------+-------+-------+----------+--------+-------------
	 public | test1 | table | postgres | 275 MB |
	 public | test2 | table | postgres | 275 MB |
	 public | test3 | table | postgres | 275 MB |
	 public | test4 | table | postgres | 275 MB |
	 public | test5 | table | postgres | 275 MB |
	 public | test6 | table | postgres | 275 MB |
	 public | test7 | table | postgres | 275 MB |
	 public | test8 | table | postgres | 275 MB |
	(8 rows)

	test=# \q


## 2.参数
### (1) pg_dump

	$ pg_dump --help
	pg_dump dumps a database as a text file or to other formats.

	Usage:
	  pg_dump [OPTION]... [DBNAME]

	General options:
	  -f, --file=FILENAME          output file or directory name
	  -F, --format=c|d|t|p         output file format (custom, directory, tar,
	                               plain text (default))
	  -j, --jobs=NUM               use this many parallel jobs to dump
	  -v, --verbose                verbose mode
	  -V, --version                output version information, then exit
	  -Z, --compress=0-9           compression level for compressed formats
	  --lock-wait-timeout=TIMEOUT  fail after waiting TIMEOUT for a table lock
	  -?, --help                   show this help, then exit

	Options controlling the output content:
	  -a, --data-only              dump only the data, not the schema
	  -b, --blobs                  include large objects in dump
	  -c, --clean                  clean (drop) database objects before recreating
	  -C, --create                 include commands to create database in dump
	  -E, --encoding=ENCODING      dump the data in encoding ENCODING
	  -n, --schema=SCHEMA          dump the named schema(s) only
	  -N, --exclude-schema=SCHEMA  do NOT dump the named schema(s)
	  -o, --oids                   include OIDs in dump
	  -O, --no-owner               skip restoration of object ownership in
	                               plain-text format
	  -s, --schema-only            dump only the schema, no data
	  -S, --superuser=NAME         superuser user name to use in plain-text format
	  -t, --table=TABLE            dump the named table(s) only
	  -T, --exclude-table=TABLE    do NOT dump the named table(s)
	  -x, --no-privileges          do not dump privileges (grant/revoke)
	  --binary-upgrade             for use by upgrade utilities only
	  --column-inserts             dump data as INSERT commands with column names
	  --disable-dollar-quoting     disable dollar quoting, use SQL standard quoting
	  --disable-triggers           disable triggers during data-only restore
	  --exclude-table-data=TABLE   do NOT dump data for the named table(s)
	  --if-exists                  use IF EXISTS when dropping objects
	  --inserts                    dump data as INSERT commands, rather than COPY
	  --no-security-labels         do not dump security label assignments
	  --no-synchronized-snapshots  do not use synchronized snapshots in parallel jobs
	  --no-tablespaces             do not dump tablespace assignments
	  --no-unlogged-table-data     do not dump unlogged table data
	  --quote-all-identifiers      quote all identifiers, even if not key words
	  --section=SECTION            dump named section (pre-data, data, or post-data)
	  --serializable-deferrable    wait until the dump can run without anomalies
	  --use-set-session-authorization
	                               use SET SESSION AUTHORIZATION commands instead of
	                               ALTER OWNER commands to set ownership

	Connection options:
	  -d, --dbname=DBNAME      database to dump
	  -h, --host=HOSTNAME      database server host or socket directory
	  -p, --port=PORT          database server port number
	  -U, --username=NAME      connect as specified database user
	  -w, --no-password        never prompt for password
	  -W, --password           force password prompt (should happen automatically)
	  --role=ROLENAME          do SET ROLE before dump

	If no database name is supplied, then the PGDATABASE environment
	variable value is used.

	Report bugs to <pgsql-bugs@postgresql.org>.

### (2) pg_restore

	$ pg_restore --help
	pg_restore restores a PostgreSQL database from an archive created by pg_dump.

	Usage:
	  pg_restore [OPTION]... [FILE]

	General options:
	  -d, --dbname=NAME        connect to database name
	  -f, --file=FILENAME      output file name
	  -F, --format=c|d|t       backup file format (should be automatic)
	  -l, --list               print summarized TOC of the archive
	  -v, --verbose            verbose mode
	  -V, --version            output version information, then exit
	  -?, --help               show this help, then exit

	Options controlling the restore:
	  -a, --data-only              restore only the data, no schema
	  -c, --clean                  clean (drop) database objects before recreating
	  -C, --create                 create the target database
	  -e, --exit-on-error          exit on error, default is to continue
	  -I, --index=NAME             restore named index
	  -j, --jobs=NUM               use this many parallel jobs to restore
	  -L, --use-list=FILENAME      use table of contents from this file for
	                               selecting/ordering output
	  -n, --schema=NAME            restore only objects in this schema
	  -O, --no-owner               skip restoration of object ownership
	  -P, --function=NAME(args)    restore named function
	  -s, --schema-only            restore only the schema, no data
	  -S, --superuser=NAME         superuser user name to use for disabling triggers
	  -t, --table=NAME             restore named table
	  -T, --trigger=NAME           restore named trigger
	  -x, --no-privileges          skip restoration of access privileges (grant/revoke)
	  -1, --single-transaction     restore as a single transaction
	  --disable-triggers           disable triggers during data-only restore
	  --if-exists                  use IF EXISTS when dropping objects
	  --no-data-for-failed-tables  do not restore data of tables that could not be
	                               created
	  --no-security-labels         do not restore security labels
	  --no-tablespaces             do not restore tablespace assignments
	  --section=SECTION            restore named section (pre-data, data, or post-data)
	  --use-set-session-authorization
	                               use SET SESSION AUTHORIZATION commands instead of
	                               ALTER OWNER commands to set ownership

	Connection options:
	  -h, --host=HOSTNAME      database server host or socket directory
	  -p, --port=PORT          database server port number
	  -U, --username=NAME      connect as specified database user
	  -w, --no-password        never prompt for password
	  -W, --password           force password prompt (should happen automatically)
	  --role=ROLENAME          do SET ROLE before restore

	The options -I, -n, -P, -t, -T, and --section can be combined and specified
	multiple times to select multiple objects.

	If no input file name is supplied, then standard input is used.

	Report bugs to <pgsql-bugs@postgresql.org>.

可以看到pg_dump和pg_restore都有-j, --jobs=NUM参数，可以使用多线程并行。这个功能是从9.3开始支持的，具体相见release note：http://www.postgresql.org/docs/9.4/interactive/release-9-3.html

## 3.pg_dump备份数据对比
### (1) 单线程

	$ rm -fr test; mkdir -p test; date '+%F %T'; pg_dump -U postgres -d test -F d -f test; date '+%F %T'
	2015-11-05 17:10:04
	2015-11-05 17:10:44
	$  ps aux | grep pg_dump | grep -v grep
	wenhang+ 50634  116  0.0 118424  2996 pts/1    R+   17:10   0:02 pg_dump -U postgres -d test -F d -f test

这条命令的意思是先删除test文件夹；再创建test文件夹；打印当前时间；使用pg_dump备份数据库到test目录，-F d是指定备份格式是目录；再打印当前时间。这两个时间差就是备份需要的时间1分40秒。

### (2) 多线程

	$ rm -fr test; mkdir -p test; date '+%F %T'; pg_dump -U postgres -d test -j 8 -F d -f test; date '+%F %T'
	2015-11-05 17:09:00
	2015-11-05 17:09:14
	$  ps aux | grep pg_dump | grep -v grep
	wenhang+ 50546  0.0  0.0 118100  2668 pts/1    S+   17:09   0:00 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50548 85.9  0.0 120516  4136 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50549 84.8  0.0 120516  4200 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50550 86.3  0.0 120516  4264 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50551 84.8  0.0 120516  4324 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50552 87.3  0.0 120516  4344 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50553 86.8  0.0 121992  5820 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50554 86.5  0.0 120516  4392 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test
	wenhang+ 50556 85.7  0.0 120516  4208 pts/1    R+   17:09   0:15 pg_dump -U postgres -d test -j 8 -F d -f test

这里加了-j 8参数，使用8线程并行，只用了14秒，可见相差了很多倍。

使用目录格式备份的原因是并行备份只支持目录格式，否则会出现如下错误：

	pg_dump: parallel backup only supported by the directory format

## 4.pg_restore恢复数据对比
### (1) 单线程

	$ psql -U postgres -c 'drop database if exists test;'; psql -U postgres -c 'create database test;'; date '+%F %T'; pg_restore -U postgres -d test -F d test ; date '+%F %T'
	DROP DATABASE
	CREATE DATABASE
	2015-11-05 17:11:36
	2015-11-05 17:13:20
	$ ps aux | grep pg_restore | grep -v grep
	wenhang+ 50758 26.0  0.0 116740  1200 pts/1    S+   17:11   0:00 pg_restore -U postgres -d test -F d test

这条命令的意思是先删除数据库test；再创建数据库test；打印当前时间；从test目录恢复数据库test；-F d是指定恢复格式是目录；再打印当前时间。这两个时间差就是恢复需要的时间差1分44秒。

### (2) 多线程

	$ psql -U postgres -c 'drop database if exists test;'; psql -U postgres -c 'create database test;'; date '+%F %T'; pg_restore -U postgres -d test -j 8 -F d test ; date '+%F %T'
	DROP DATABASE
	CREATE DATABASE
	2015-11-05 17:26:22
	2015-11-05 17:26:56
	$ ps aux | grep pg_restore | grep -v grep
	wenhang+ 53848  0.0  0.0 116600  1264 pts/1    S+   17:26   0:00 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53850 11.7  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53851 11.9  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53852 12.1  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53853 12.2  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53854 12.0  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53855 12.6  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53857 11.6  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test
	wenhang+ 53858 12.2  0.0 116744   976 pts/1    S+   17:26   0:02 pg_restore -U postgres -d test -j 8 -F d test

可见多线程恢复也快了不少，如果io更快的话，这个提升会更明显
