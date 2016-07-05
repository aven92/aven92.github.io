---
layout: post
title: PostgreSQL备份之pg_basebackup
categories: PostgreSQL
---

<!--more-->

## 1.准备

连接上master节点创建replication用户

    # CREATE ROLE replicator with login replication password 123456；

修改白名单pg_hba.conf

    # host    replication    all        xxxxx/32    md5

## 2.制作基础备份

    # pg_basebackup --format=tar \ #使用打包方式备份
    --xlog-method=fetch \    #获取wal文件方式，等备份结束后把备份期间所有的wal备份
    --compress=1 \    #压缩等级
    --checkpoint=fast \    #执行检查点的方式
    --label=backup \    #标签
    --pgdata=/data/backup/ #备份目录
    --progress \    #打印过程
    --verbose \    #打印详细信息
    --host='172.17.5.45' \    #master节点ip
    --port=5432 \    #master节点端口
    --user=postgres    #连接master节点的用户
    Password: 
    transaction log start point: 2/34000028 on timeline 1
    414272/414272 kB (100%), 3/3 tablespaces                                         
    transaction log end point: 2/34000128
    pg_basebackup: base backup completed

备份默认会占用一个max_wal_senders连接，需要设置大一点，还可以使用下面的参数限制传输速度

    -r, --max-rate=RATE    maximum transfer rate to transfer data directory
                            (in kB/s, or use suffix "k" or "M")

    # ll backup/
    total 123196
    -rw-r--r--. 1 root root       129 Sep  8 01:21 25375.tar.gz
    -rw-r--r--. 1 root root       128 Sep  8 01:21 25376.tar.gz
    -rw-r--r--. 1 root root 126141406 Sep  8 01:22 base.tar.gz

base.tar.gz是数据目录的备份，其他两个是表空间的备份

## 3.恢复
### (1) 解压缩

    # cd /data/backup/
    # mkdir pgdata 25375 25376
    # tar zxf base.tar.gz -C pgdata/
    # ls pgdata/
    backup_label  global   pg_dynshmem  pg_ident.conf  pg_logical    pg_notify    pg_serial     pg_stat      pg_subtrans  pg_twophase  pg_xlog               postgresql.conf
    base          pg_clog  pg_hba.conf  pg_log         pg_multixact  pg_replslot  pg_snapshots  pg_stat_tmp  pg_tblspc    PG_VERSION   postgresql.auto.conf  recovery.conf
    # tar zxf 25375.tar.gz -C 25375
    # tar zxf 25376.tar.gz -C 25376 
    # ls
    25375  25375.tar.gz  25376  25376.tar.gz  base.tar.gz  pgdata

### (2) 修改表空间软连接

    # ln -sf /data/backup/25375 pgdata/pg_tblspc/25375 [root@node2 backup]# ln -sf /data/backup/25376 pgdata/pg_tblspc/25376                     [root@node2 backup]# ll pgdata/pg_tblspc/ total 0 lrwxrwxrwx. 1 root root 18 Sep  8 01:35 25375 -> /data/backup/25375 lrwxrwxrwx. 1 root root 18 Sep  8 01:35 25376 -> /data/backup/25376

### (3) 修改数据目录权限

    # chown -R postgres:postgres pgdata/ 25375 25376 
    # chmod 0700 pgdata/
    # ll
    total 123208
    drwxr-xr-x.  3 postgres postgres      4096 Sep  8 01:32 25375
    -rw-r--r--.  1 root     root           129 Sep  8 01:21 25375.tar.gz
    drwxr-xr-x.  3 postgres postgres      4096 Sep  8 01:33 25376
    -rw-r--r--.  1 root     root           128 Sep  8 01:21 25376.tar.gz
    -rw-r--r--.  1 root     root     126141406 Sep  8 01:22 base.tar.gz
    drwx------. 19 postgres postgres      4096 Sep  8 01:31 pgdata

### (4) 启动

    # su postgres
    $ pg_ctl -D pgdata/ start
    server starting
     [    2015-09-08 05:40:04.510 UTC 11107 55ee74b4.2b63 1 0 ]LOG:  redirecting log output to logging collector process
    [    2015-09-08 05:40:04.511 UTC 11107 55ee74b4.2b63 2 0 ]HINT:  Future log output will appear in directory "pg_log".
    $ psql 
    psql (9.4.4)
    Type "help" for help.
    
    postgres=# \db+
                                      List of tablespaces
        Name    |  Owner   |      Location      | Access privileges | Options | Description 
    ------------+----------+--------------------+-------------------+---------+-------------
     pg_default | postgres |                    |                   |         | 
     pg_global  | postgres |                    |                   |         | 
     tblspc1    | postgres | /data/backup/25375 |                   |         | 
     tblspc2    | postgres | /data/backup/25376 |                   |         | 
    (4 rows)

## 4.制作从库
### (1) 获取备份文件

    # pg_basebackup --format=plain \    #使用目录方式
    --write-recovery-conf \    #生成简单recovery.conf
    --xlog-method=stream \    #使用stream的方式备份wal文件，会再占用一个max_wal_sender,wal文件一生成就会备份
    --checkpoint=fast \
    --label=backup \
    --pgdata=/data/test_pgdata_94 \
    --progress \
    --verbose \
    --host='172.17.5.45' \
    --port=5432 \
    --user=postgres
    Password: 
    transaction log start point: 2/38000028 on timeline 1
    pg_basebackup: starting background WAL receiver
    397887/397887 kB (100%), 3/3 tablespaces                                         
    transaction log end point: 2/380000F0
    pg_basebackup: waiting for background process to finish streaming ...
    pg_basebackup: base backup completed

### (2) 修改目录权限

    # chown -R postgres:postgres test_pgdata_94 tblspc1/ tblspc2/
    # chmod 0700 test_pgdata_94
    # ll test_pgdata_94
    total 104
    -rw-------. 1 postgres postgres  189 Sep  8 01:43 backup_label
    drwx------. 6 postgres postgres 4096 Sep  8 01:43 base
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 global
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_clog
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_dynshmem
    -rw-------. 1 postgres postgres 4581 Sep  8 01:43 pg_hba.conf
    -rw-------. 1 postgres postgres 1636 Sep  8 01:43 pg_ident.conf
    drwxr-xr-x. 2 postgres postgres 4096 Sep  8 01:43 pg_log
    drwx------. 4 postgres postgres 4096 Sep  8 01:43 pg_logical
    drwx------. 4 postgres postgres 4096 Sep  8 01:43 pg_multixact
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_notify
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_replslot
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_serial
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_snapshots
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_stat
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_stat_tmp
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_subtrans
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_tblspc
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 pg_twophase
    -rw-------. 1 postgres postgres    4 Sep  8 01:43 PG_VERSION
    drwx------. 3 postgres postgres 4096 Sep  8 01:43 pg_xlog
    -rw-------. 1 postgres postgres   88 Sep  8 01:43 postgresql.auto.conf
    -rw-------. 1 postgres postgres 4676 Sep  8 01:43 postgresql.conf
    -rw-r--r--. 1 postgres postgres  131 Sep  8 01:43 recovery.conf
    # ll tblspc1/
    total 4
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 PG_9.4_201409291
    # ll tblspc2
    total 4
    drwx------. 2 postgres postgres 4096 Sep  8 01:43 PG_9.4_201409291

### (3) 启动

    # su postgres
    $ pg_ctl -D test_pgdata_94 start
    server starting
    $ [    2015-09-08 05:49:55.383 UTC 12100 55ee7703.2f44 1 0 ]LOG:  redirecting log output to logging collector process
    [    2015-09-08 05:49:55.383 UTC 12100 55ee7703.2f44 2 0 ]HINT:  Future log output will appear in directory "pg_log".

    $ psql 
    psql (9.4.4)
    Type "help" for help.

    postgres=# \db+
                                    List of tablespaces
        Name    |  Owner   |   Location    | Access privileges | Options | Description 
    ------------+----------+---------------+-------------------+---------+-------------
     pg_default | postgres |               |                   |         | 
     pg_global  | postgres |               |                   |         | 
     tblspc1    | postgres | /data/tblspc1 |                   |         | 
     tblspc2    | postgres | /data/tblspc2 |                   |         | 
    (4 rows)

参考：http://www.postgresql.org/docs/9.4/static/app-pgbasebackup.html
