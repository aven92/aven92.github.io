---
layout: post
title: PostgreSQL同步复制故障测试
categories: PostgreSQL
---

<!--more-->

## 1.备用同步复制节点down了

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

    postgres=# create table test1 ( id bigint );
    CREATE TABLE
    postgres=# \d test1
        Table "public.test1"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

在mastrer上查看是正常工作的，现在把172.17.5.47节点停机

    pg_ctl -D /data/pg940_data/ -mf stop

在master上验证

    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12263 |       10 | postgres | node2            | 172.17.5.46 |                 |       35271 | 2015-09-06 14:44:52.677004+08 |       400593 | streaming | 1/74001678    | 1/74001678     | 1/
    74001678     | 1/74001678      |             1 | sync
    (1 row)

    postgres=# create table test2 ( id bigint ); 
    CREATE TABLE
    postgres=# \d test2
        Table "public.test2"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

master节点和同步复制slave节点可以正常工作

## 2.同步复制slave节点down了

    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12377 |       10 | postgres | node3            | 172.17.5.47 |                 |       55722 | 2015-09-06 15:10:02.882202+08 |       400594 | streaming | 1/740029F0    | 1/740029F0     | 1/
    740029F0     | 1/740029F0      |             2 | potential
     12263 |       10 | postgres | node2            | 172.17.5.46 |                 |       35271 | 2015-09-06 14:44:52.677004+08 |       400594 | streaming | 1/740029F0    | 1/740029F0     | 1/
    740029F0     | 1/740029F0      |             1 | sync
    (2 rows)

    postgres=# create table test3 ( id bigint  );
    CREATE TABLE
    postgres=# \d test3
        Table "public.test3"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

在mastrer上查看是正常工作的，现在把172.17.5.46节点停机

    pg_ctl -D /data/pg940_data/ -mf stop

在master上验证

    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12377 |       10 | postgres | node3            | 172.17.5.47 |                 |       55722 | 2015-09-06 15:10:02.882202+08 |       400595 | streaming | 1/74004C10    | 1/74004C10     | 1/
    74004C10     | 1/74004C10      |             2 | sync
    (1 row)
    postgres=# create table test4 ( id bigint  );  
    CREATE TABLE
    postgres=# \d test4
        Table "public.test4"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

可以看出同步复制节点切换到了node3，并且是可以正常工作的

## 3.slave节点都down掉了

    postgres=# select * from pg_stat_replication ;
      pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | fl
    ush_location | replay_location | sync_priority | sync_state 
    -------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+---
    -------------+-----------------+---------------+------------
     12377 |       10 | postgres | node3            | 172.17.5.47 |                 |       55722 | 2015-09-06 15:10:02.882202+08 |       400596 | streaming | 1/74005F38    | 1/74005F38     | 1/
    74005F38     | 1/74005F38      |             2 | sync
    (1 row)

    postgres=# create table test5 ( id bigint  );
    CREATE TABLE
    postgres=# \d test5
        Table "public.test5"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

在master上查看是正常工作的，现在把172.17.5.47节点也停机

    pg_ctl -D /data/pg940_data/ -mf stop

在master上验证

    postgres=# select * from pg_stat_replication ;
     pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | backend_xmin | state | sent_location | write_location | flush_location | replay_l
    ocation | sync_priority | sync_state 
    -----+----------+---------+------------------+-------------+-----------------+-------------+---------------+--------------+-------+---------------+----------------+----------------+---------
    --------+---------------+------------
    (0 rows)

    postgres=# create table test6 ( id bigint  );

master上不能更新了，会一直等待，直到synchronous_standby_names中的一个节点连接上master节点，但是可以ctrl+c取消，那样会进行本地提交

    ^CCancel request sent
    WARNING:  canceling wait for synchronous replication due to user request
    DETAIL:  The transaction has already committed locally, but might not have been replicated to the standby.
    CREATE TABLE
    postgres=# \d test6 
        Table "public.test6"
     Column |  Type  | Modifiers 
    --------+--------+-----------
     id     | bigint |

## 4.结论

在同步复制中：

    同步的slave节点down掉会将后面的节点切换为同步节点；

    非同步slave节点down掉不会进行切换；

    如果slave节点没有全部down掉，不会影响master的使用，如果都down掉的话，master则不能进行提交。
