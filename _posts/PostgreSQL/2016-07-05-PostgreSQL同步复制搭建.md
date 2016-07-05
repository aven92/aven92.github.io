---
layout: post
title: PostgreSQL同步复制搭建
categories: PostgreSQL
---

<!--more-->

## 1.初始化master节点
### (1) 安装PostgreSQL
### (2) 初始化db

    initdb -D /data/pg940_data


## 2.配置master节点
### (1) 修改postgresql.conf

    $ vim /data/pg940_data/postgresql.conf
    wal_level = hot_standby
    fsync = on
    synchronous_commit = on    #同步复制必须设置为on
    wal_sync_method = fdatasync
    max_wal_senders = 4        #配置的是两个slave节点，最好设置大一点，方便以后添加slave节点，pg_basebackup使用stream方式需要占用2个槽
    wal_keep_segments = 100

### (2) 修改白名单

    $ vim /data/pg940_data/pg_hba.conf
    host    all                 all             172.17.0.0/16           md5
    host    replication         all             172.17.0.0/16           md5


### (3) 启动master节点

    $ pg_ctl -D /data/pg940_data start

## 3.制作slave节点
### (1) 获取master节点的基础备份

    $ sudo mkdir -p /data/pg940_data
    $ sudo chown -R postgres:postgres /data/pg940_data
    $ pg_basebackup -X s -h 172.17.5.45 -U postgres -D /data/pg940_data

### (2) 配置postgresql.conf

    hot_standby = on
    max_standby_archive_delay = 30s
    max_standby_streaming_delay = 30s

### (3) 配置recovery.conf

    standby_mode = 'on'
    primary_conninfo = 'host=172.17.5.45 port=5432 user=postgres password=xxxxxx application_name=node2'
    restore_command = ''
    archive_cleanup_command = ''

### (4) 启动slave节点

    $ pg_ctl -D /data/pg940_data start
    $ psql -U postgres
    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12262 |       10 | postgres | node3            | 172.17.5.47 |                 |       48691 | 2015-09-06 14:44:52.674976+08 |       400592 | streaming | 1/74000210    | 1/74000210     | 1/
    74000210     | 1/74000210      |             0 | async
     12263 |       10 | postgres | node2            | 172.17.5.46 |                 |       35271 | 2015-09-06 14:44:52.677004+08 |       400592 | streaming | 1/74000210    | 1/74000210     | 1/
    74000210     | 1/74000210      |             0 | async
    (2 rows)


可以看出两个节点现在都还是异步的

## 4.修改为同步复制

    $ vim /data/pg940_data/postgresql.conf
    synchronous_standby_names = 'node2, node3'    # reload配置就行，不需要重启
    $ psql -U postgres
    psql (9.4.4)
    Type "help" for help.
    postgres=# show synchronous_standby_names;
     synchronous_standby_names 
    ---------------------------
     
    (1 row)
    postgres=# select pg_reload_conf();
     pg_reload_conf 
    ----------------
     t
    (1 row)
    postgres=# show synchronous_standby_names;
     synchronous_standby_names 
    ---------------------------
     node2, node3
    (1 row)
    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12262 |       10 | postgres | node3            | 172.17.5.47 |                 |       48691 | 2015-09-06 14:44:52.674976+08 |       400592 | streaming | 1/740002E8    | 1/740002E8     | 1/
    740002E8     | 1/740002E8      |             2 | potential
     12263 |       10 | postgres | node2            | 172.17.5.46 |                 |       35271 | 2015-09-06 14:44:52.677004+08 |       400592 | streaming | 1/740002E8    | 1/740002E8     | 1/
    740002E8     | 1/740002E8      |             1 | sync
    (2 rows)

可以看出现在node2是同步复制的，node3是备用同步复制节点
