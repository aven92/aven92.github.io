---
layout: post
title: Pacemaker+Corosync+PostgreSQL集群故障测试和解决方案
categories: PostgreSQL
---

<!--more-->

## 1.环境

	$ cat /etc/redhat-release 
	CentOS Linux release 7.0.1406 (Core) 
	$ uname -a
	Linux zhaopin-5-90 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 11:02:08 2015Last change: Tue Oct 20 10:57:37 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

node1: 172.17.5.90
node2: 172.17.5.91
node3: 172.17.5.92
node4: 172.17.5.93 应急

vip-master: 172.17.5.99
vip-slave:  172.17.5.98


## 2.PostgreSQL故障
### (1) 单节点故障

1）master故障

在node3上执行：

	$ ps aux | grep postgres | grep -v grep
	postgres  29482  0.0  0.2 103216  7900 ?        S    10:57   0:00 /usr/bin/postgres -D /data/postgresql/data -c config_file=/data/postgresql/data/postgresql.conf
	postgres  29524  0.0  0.0 103216  1980 ?        Ss   10:57   0:00 postgres: checkpointer process   
	postgres  29525  0.0  0.0 103216  1988 ?        Ss   10:57   0:00 postgres: writer process   
	postgres  29526  0.0  0.0 103216  1896 ?        Ss   10:57   0:00 postgres: wal writer process   
	postgres  29527  0.0  0.0 104044  3008 ?        Ss   10:57   0:00 postgres: autovacuum launcher process   
	postgres  29528  0.0  0.0  88912  1792 ?        Ss   10:57   0:00 postgres: archiver process   last was 00000001000000000000000E
	postgres  29529  0.0  0.0  89024  1920 ?        Ss   10:57   0:00 postgres: stats collector process   
	postgres  29790  0.0  0.0 104060  3208 ?        Ss   10:57   0:00 postgres: wal sender process postgres 172.17.5.90(35734) streaming 0/F0000E0
	postgres  29791  0.0  0.0 104192  3392 ?        Ss   10:57   0:00 postgres: wal sender process postgres 172.17.5.91(60722) streaming 0/F0000E0
	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 29790 |       10 | postgres | zhaopin-5-90     | 172.17.5.90 |                 |       35734 | 2015-10-20 02:57:25.718047+00 | streaming | 0/F0000E0     | 0/F0000E0      | 0/F0000E0      | 0/F0000E0       |             1 | sync
	 29791 |       10 | postgres | zhaopin-5-91     | 172.17.5.91 |                 |       60722 | 2015-10-20 02:57:25.718566+00 | streaming | 0/F0000E0     | 0/F0000E0      | 0/F0000E0      | 0/F0000E0       |             2 | potential
	(2 rows)
	postgres=# \q

杀掉PostgreSQL的进程

	$ sudo killall postgres
	$ ps aux | grep postgres | grep -v grep

PostgreSQL的进程已经没有了

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 11:11:54 2015Last change: Tue Oct 20 11:10:31 2015 by root via crm_attribute on zhaopin-5-90
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-90 ]
	     Slaves: [ zhaopin-5-91 ]
	     Stopped: [ zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	Failed Actions:
	* pgsql_start_0 on zhaopin-5-92 'unknown error' (1): call=33, status=complete, exitreason='My data may be inconsistent. You have to remove /var/lib/pgsql/tmp/PGSQL.lock file to force start.',
	    last-rc-change='Tue Oct 20 11:10:14 2015', queued=0ms, exec=389ms
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

查看集群状态可见，master和vip-master已经切换到node1上了


在node1上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 59835 |       10 | postgres | zhaopin-5-91     | 172.17.5.98 |                 |       46393 | 2015-10-20 03:10:24.392231+00 | streaming | 0/F000140     | 0/F000140      | 0/F000140      | 0/F000140       |             1 | sync
	(1 row)
	postgres=# create table test ( id bigint );
	CREATE TABLE
	postgres=# \dt test
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | test | table | postgres
	(1 row)
	postgres=# \q

可以正常提供服务


恢复node3为新master node1的slave,在node3上执行

	$ sudo su - postgres
	Last login: Tue Oct 20 11:10:16 CST 2015
	$ rm -f /var/lib/pgsql/tmp/PGSQL.lock
	$ rm -fr /data/postgresql/data/*
	$ pg_basebackup -h 172.17.5.90 -U postgres -D /data/postgresql/data/ -X stream -P
	20243/20243 kB (100%), 1/1 tablespace
	$ logout
	$ sudo pcs resource cleanup pgsql-master
	Resource: pgsql-master successfully cleaned up
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 11:19:45 2015Last change: Tue Oct 20 11:19:45 2015 by root via crm_attribute on zhaopin-5-90
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-90 ]
	     Slaves: [ zhaopin-5-91 zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

可见node3已经加到集群里面了。如果配置archive_command和restore_command的话，就不用删除数据目录，按照下面pacemaker故障的解决方法就行。


2）slave故障

在node2上执行:

	$ ps aux | grep postgres | grep -v grep
	postgres  35611  0.0  0.2 103216  7876 ?        S    10:57   0:00 /usr/bin/postgres -D /data/postgresql/data -c config_file=/data/postgresql/data/postgresql.conf
	postgres  35637  0.0  0.0 103280  2392 ?        Ss   10:57   0:00 postgres: startup process   recovering 000000010000000000000011
	postgres  35656  0.0  0.0 103216  2144 ?        Ss   10:57   0:00 postgres: checkpointer process   
	postgres  35657  0.0  0.0 103216  1648 ?        Ss   10:57   0:00 postgres: writer process   
	postgres  35658  0.0  0.0  88912  1512 ?        Ss   10:57   0:00 postgres: stats collector process   
	postgres  55111  0.1  0.0 110228  3364 ?        Ss   11:10   0:00 postgres: wal receiver process   streaming 0/12000000
	$ sudo killall postgres
	$ ps aux | grep postgres | grep -v grep
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 11:22:57 2015Last change: Tue Oct 20 11:22:53 2015 by root via crm_attribute on zhaopin-5-90
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-90 ]
	     Slaves: [ zhaopin-5-91 zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	Failed Actions:
	* pgsql_monitor_4000 on zhaopin-5-91 'unknown error' (1): call=47, status=complete, exitreason='none',
	    last-rc-change='Tue Oct 20 11:22:51 2015', queued=0ms, exec=0ms
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

可见vip-slave自动切换到了node3


在node1上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# create table test ( id bigint );
	ERROR:  relation "test" already exists
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |        backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 99082 |       10 | postgres | zhaopin-5-91     | 172.17.5.91 |                 |       60726 | 2015-10-20 03:22:55.84451+00 | streaming | 0/120000B8    | 0/120000B8     | 0/120000B8     | 0/120000B8      |             2 | potential
	 88344 |       10 | postgres | zhaopin-5-92     | 172.17.5.92 |                 |       60891 | 2015-10-20 03:19:43.1618+00  | streaming | 0/120000B8    | 0/120000B8     | 0/120000B8     | 0/120000B8      |             1 | sync
	(2 rows)
	postgres=# \q

可见同步复制节点切换到了node3上


### (2) 两节点故障

1）master和slave故障

在node1和node2执行:

	$ sudo killall postgres
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 13:47:01 2015Last change: Tue Oct 20 13:47:00 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Stopped: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Stopped
	Failed Actions:
	* pgsql_start_0 on zhaopin-5-90 'unknown error' (1): call=50, status=complete, exitreason='My data may be inconsistent. You have to remove /var/lib/pgsql/tmp/PGSQL.lock file to force start.',
	    last-rc-change='Tue Oct 20 13:46:53 2015', queued=0ms, exec=285ms
	* pgsql_start_0 on zhaopin-5-91 'unknown error' (1): call=34, status=complete, exitreason='My data may be inconsistent. You have to remove /var/lib/pgsql/tmp/PGSQL.lock file to force start.',
	    last-rc-change='Tue Oct 20 13:46:33 2015', queued=0ms, exec=343ms
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

可见node1和node2停机了,node3自动切成了master,vip-slave也失效了


在node3上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	 pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | state | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-----+----------+---------+------------------+-------------+-----------------+-------------+---------------+-------+---------------+----------------+----------------+-----------------+---------------+------------
	(0 rows)
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | test | table | postgres
	(1 row)
	postgres=# drop table test ;
	DROP TABLE
	postgres=# \q

可见master可以正常提供服务


将两个节点变成新master node3的slave,解决方法如上面单节点故障->master节点的一样,结果如下:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 13:53:27 2015Last change: Tue Oct 20 13:53:16 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

2）两个slave故障

在node1和node2执行:

	$ sudo killall postgres
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 13:57:23 2015Last change: Tue Oct 20 13:57:20 2015 by hacluster via crmd on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

可见集群会自动恢复


## 3.Pacemaker故障
### (1) 单节点故障

1）停掉Pacemaker服务

在node1上执行:

	$ systemctl status pacemaker
	pacemaker.service - Pacemaker High Availability Cluster Manager
	   Loaded: loaded (/usr/lib/systemd/system/pacemaker.service; disabled)
	   Active: active (running) since Tue 2015-10-20 13:45:28 CST; 15min ago
	 Main PID: 35622 (pacemakerd)
	   CGroup: /system.slice/pacemaker.service
	           ├─35622 /usr/sbin/pacemakerd -f
	           ├─35623 /usr/libexec/pacemaker/cib
	           ├─35624 /usr/libexec/pacemaker/stonithd
	           ├─35625 /usr/libexec/pacemaker/lrmd
	           ├─35626 /usr/libexec/pacemaker/attrd
	           ├─35627 /usr/libexec/pacemaker/pengine
	           └─35628 /usr/libexec/pacemaker/crmd
	$ sudo systemctl stop pacemaker
	$ systemctl status pacemaker
	pacemaker.service - Pacemaker High Availability Cluster Manager
	   Loaded: loaded (/usr/lib/systemd/system/pacemaker.service; disabled)
	   Active: inactive (dead)
	$ sudo pcs status
	Error: cluster is not currently running on this node

2）查看状态

在node2上执行:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:02:41 2015Last change: Tue Oct 20 14:01:06 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 ]
	OFFLINE: [ zhaopin-5-90 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-91 ]
	     Stopped: [ zhaopin-5-90 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  172.17.5.90: Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

可见node1的集群和PostgreSQL都宕机了,vip-slave切换到了node2上


3）验证可用性

在node3上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 78298 |       10 | postgres | zhaopin-5-91     | 172.17.5.91 |                 |       60755 | 2015-10-20 05:57:08.882432+00 | streaming | 0/16000150    | 0/16000150     | 0/16000150     | 0/16000150      |             1 | sync
	(1 row)
	postgres=# create table test ( id int );
	CREATE TABLE
	postgres=# \q

可见集群仍然可用


4）解决方案

在node1上执行:

	$ sudo pcs cluster start
	Starting Cluster...
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:07:54 2015Last change: Tue Oct 20 14:07:51 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

在node3上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid   | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	--------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 110962 |       10 | postgres | zhaopin-5-90     | 172.17.5.90 |                 |       35910 | 2015-10-20 06:07:51.026655+00 | streaming | 0/16013738    | 0/16013738     | 0/16013738     | 0/16013738      |             2 | potential
	  78298 |       10 | postgres | zhaopin-5-91     | 172.17.5.91 |                 |       60755 | 2015-10-20 05:57:08.882432+00 | streaming | 0/16013738    | 0/16013738     | 0/16013738     | 0/16013738      |             1 | sync
	(2 rows)
	postgres=# \q

可见已经恢复


### (2) 两节点故障

1）停掉Pacemaker服务

在node1和node3上执行:

	$ sudo systemctl stop pacemaker

2）查看状态

在node2上执行:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:11:34 2015Last change: Tue Oct 20 14:10:59 2015 by root via crm_attribute on zhaopin-5-91
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-91 ]
	OFFLINE: [ zhaopin-5-90 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-91 ]
	     Stopped: [ zhaopin-5-90 zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Stopped
	PCSD Status:
	  172.17.5.90: Online
	  zhaopin-5-91 (172.17.5.91): Online
	  172.17.5.92: Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled


3）验证可用性

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	 pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | state | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-----+----------+---------+------------------+-------------+-----------------+-------------+---------------+-------+---------------+----------------+----------------+-----------------+---------------+------------
	(0 rows)
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | test | table | postgres
	(1 row)
	postgres=# drop table test ;
	DROP TABLE
	postgres=# \q

可见集群仍然可用


4）解决方案

对于原来的slave节点故障如上面单节点故障处理方法一样，原master节点操作如下:

	$ sudo rm -f /var/lib/pgsql/tmp/PGSQL.lock
	$ sudo pcs cluster start 
	Starting Cluster...
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:17:32 2015Last change: Tue Oct 20 14:17:31 2015 by root via crm_attribute on zhaopin-5-91
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-91 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-90
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled


在node2上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 54687 |       10 | postgres | zhaopin-5-90     | 172.17.5.98 |                 |       40297 | 2015-10-20 06:17:02.084825+00 | streaming | 0/17005B08    | 0/17005B08     | 0/17005B08     | 0/17005B08      |             1 | sync
	 56147 |       10 | postgres | zhaopin-5-92     | 172.17.5.92 |                 |       32904 | 2015-10-20 06:17:29.874711+00 | streaming | 0/17005B08    | 0/17005B08     | 0/17005B08     | 0/17005B08      |             2 | potential
	(2 rows)
	postgres=# \q


## 4.Corosync故障
### (1) 单节点故障

1）停掉Corosync服务

在node1上执行:

	$ systemctl status corosync.service 
	corosync.service - Corosync Cluster Engine
	   Loaded: loaded (/usr/lib/systemd/system/corosync.service; disabled)
	   Active: active (running) since Tue 2015-10-20 13:45:28 CST; 35min ago
	  Process: 35600 ExecStart=/usr/share/corosync/corosync start (code=exited, status=0/SUCCESS)
	 Main PID: 35607 (corosync)
	   CGroup: /system.slice/corosync.service
	           └─35607 corosync
	$ sudo systemctl stop corosync.service 
	$ systemctl status corosync.service 
	corosync.service - Corosync Cluster Engine
	   Loaded: loaded (/usr/lib/systemd/system/corosync.service; disabled)
	   Active: inactive (dead)
	$ sudo pcs status
	Error: cluster is not currently running on this node

2）查看状态

在node2上执行:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:22:36 2015Last change: Tue Oct 20 14:21:37 2015 by root via crm_attribute on zhaopin-5-91
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 ]
	OFFLINE: [ zhaopin-5-90 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-91 ]
	     Slaves: [ zhaopin-5-92 ]
	     Stopped: [ zhaopin-5-90 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	PCSD Status:
	  172.17.5.90: Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

3）验证可用性

在node2上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 56147 |       10 | postgres | zhaopin-5-92     | 172.17.5.92 |                 |       32904 | 2015-10-20 06:17:29.874711+00 | streaming | 0/17005BA0    | 0/17005BA0     | 0/17005BA0     | 0/17005BA0      |             1 | sync
	(1 row)
	postgres=# \dt
	No relations found.
	postgres=# create table test ( id bigint );
	CREATE TABLE
	postgres=# \q

可见集群仍然可用


4）解决方案

如同Pacemaker单节点故障解决方案


### (2) 两节点故障

在node1和node2上执行:

	$ sudo systemctl stop corosync.service
	$ systemctl status corosync.service
	corosync.service - Corosync Cluster Engine
	   Loaded: loaded (/usr/lib/systemd/system/corosync.service; disabled)
	   Active: inactive (dead)
	$ sudo pcs status
	Error: cluster is not currently running on this node


2）查看状态

在node3上执行:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:27:17 2015Last change: Tue Oct 20 14:26:35 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition WITHOUT quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-92 ]
	OFFLINE: [ zhaopin-5-90 zhaopin-5-91 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Stopped: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Stopped
	PCSD Status:
	  172.17.5.90: Online
	  172.17.5.91: Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

3）验证可用性

在node3上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	 pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | state | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-----+----------+---------+------------------+-------------+-----------------+-------------+---------------+-------+---------------+----------------+----------------+-----------------+---------------+------------
	(0 rows)
	postgres=# \dt
	        List of relations
	 Schema | Name | Type  |  Owner   
	--------+------+-------+----------
	 public | test | table | postgres
	(1 row)
	postgres=# drop table test ;
	DROP TABLE
	postgres=# \q

4）解决方案

如同Pacemaker故障中的两节点故障


## 5.服务器故障
### (1) 停掉机器

在node1上执行:

	$ sudo shutdown -h now

### (2) 查看状态

在node3上执行:

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 14:33:47 2015Last change: Tue Oct 20 14:33:22 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-92 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 ]
	OFFLINE: [ zhaopin-5-90 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-91 ]
	     Stopped: [ zhaopin-5-90 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  172.17.5.90: Offline
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

### (3) 验证可用性

在node3上执行:

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state 
	-------+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 15678 |       10 | postgres | zhaopin-5-91     | 172.17.5.91 |                 |       60771 | 2015-10-20 06:30:11.747191+00 | streaming | 0/18006120    | 0/18006120     | 0/18006120     | 0/18006120      |             1 | sync
	(1 row)
	postgres=# \dt
	No relations found.
	postgres=# create table test ( id bigint );
	CREATE TABLE
	postgres=# \q

可见集群仍然可用


### (4) 解决方案

在node4上执行:

	$ sudo yum install -y -q pacemaker pcs psmisc policycoreutils-python
	$ sudo setenforce 0
	$ sudo sed -i.bak "s/SELINUX=enforcing/SELINUX=permissive/g" /etc/selinux/config
	$ sudo systemctl disable firewalld.service
	$ sudo systemctl stop firewalld.service
	$ sudo iptables --flush
	$ sudo systemctl start pcsd.service
	$ sudo systemctl enable pcsd.service
	ln -s '/usr/lib/systemd/system/pcsd.service' '/etc/systemd/system/multi-user.target.wants/pcsd.service'
	$ sudo passwd hacluster
	Changing password for user hacluster.
	New password:
	Retype new password:
	passwd: all authentication tokens updated successfully.
	$ sudo mkdir -p /data/postgresql/data
	$ sudo mkdir -p /data/postgresql/xlog_archive
	$ sudo chown -R postgres:postgres /data/postgresql/
	$ sudo chmod 0700 /data/postgresql/data
	$ sudo su - postgres
	$ pg_basebackup -h 172.17.5.92 -U postgres -D /data/postgresql/data/ -X stream -P
20245/20245 kB (100%), 1/1 tablespace


在node2上执行:

	$ sudo pcs cluster auth 172.17.5.90 172.17.5.91 172.17.5.92 172.17.5.93 -u hacluster -p hacluster
	172.17.5.90: Authorized
	172.17.5.91: Authorized
	172.17.5.92: Authorized
	172.17.5.93: Authorized
	$ sudo pcs cluster node add 172.17.5.93 --start
	172.17.5.90: Corosync updated
	172.17.5.91: Corosync updated
	172.17.5.92: Corosync updated
	172.17.5.93: Succeeded
	172.17.5.93: Starting Cluster...
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 16:07:12 2015Last change: Tue Oct 20 16:01:06 2015 by hacluster via crmd on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	4 nodes and 5 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 zhaopin-5-93 ]
	OFFLINE: [ zhaopin-5-90 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-91 ]
	     Stopped: [ zhaopin-5-90 zhaopin-5-93 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  172.17.5.90: Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	  zhaopin-5-93 (172.17.5.93): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled


更新PostgreSQL集群，添加新加的节点，会有闪断

	$ sudo pcs resource update pgsql-master pgsql master-max=1 master-node-max=1 clone-max=4 clone-node-max=1 notify=true
	$ sudo pcs resource update pgsql pgsql node_list="zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 zhaopin-5-93"
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 16:13:55 2015Last change: Tue Oct 20 16:13:32 2015 by hacluster via crmd on zhaopin-5-91
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	4 nodes and 6 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 zhaopin-5-93 ]
	OFFLINE: [ zhaopin-5-90 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-93 ]
	     Slaves: [ zhaopin-5-91 zhaopin-5-92 ]
	     Stopped: [ zhaopin-5-90 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-93
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  172.17.5.90: Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	  zhaopin-5-93 (172.17.5.93): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled
	$ sudo pcs cluster node remove 172.17.5.90
	172.17.5.90: Stopping Cluster (pacemaker)...
	172.17.5.90: Successfully destroyed cluster
	172.17.5.91: Corosync updated
	172.17.5.92: Corosync updated
	172.17.5.93: Corosync updated
	$ sudo crmadmin -N
	(null) node: zhaopin-5-92 (3)
	(null) node: zhaopin-5-91 (2)
	(null) node: zhaopin-5-93 (4)
	$ sudo crm_node -R zhaopin-5-90 --force
	$ sudo crmadmin -N
	(null) node: zhaopin-5-92 (3)
	(null) node: zhaopin-5-91 (2)
	(null) node: zhaopin-5-93 (4)
	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Tue Oct 20 16:58:05 2015Last change: Tue Oct 20 16:56:06 2015 by root via crm_node on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-91 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 6 resources configured
	Online: [ zhaopin-5-91 zhaopin-5-92 zhaopin-5-93 ]
	Full list of resources:
	 Master/Slave Set: pgsql-master [pgsql]
	     Masters: [ zhaopin-5-93 ]
	     Slaves: [ zhaopin-5-91 zhaopin-5-92 ]
	 Resource Group: master-group
	     vip-master(ocf::heartbeat:IPaddr2):Started zhaopin-5-93
	 Resource Group: slave-group
	     vip-slave(ocf::heartbeat:IPaddr2):Started zhaopin-5-91
	PCSD Status:
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	  zhaopin-5-93 (172.17.5.93): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/enabled

## 6.结论

集群中还剩最后一台机器仍能提供服务
