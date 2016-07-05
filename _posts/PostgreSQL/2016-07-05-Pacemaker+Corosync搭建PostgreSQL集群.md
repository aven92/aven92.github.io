---
layout: post
title: Pacemaker+Corosync搭建PostgreSQL集群
categories: PostgreSQL
---

<!--more-->

## 1.环境 

	$ cat /etc/redhat-release 
	CentOS Linux release 7.0.1406 (Core) 
	$ uname -a
	Linux zhaopin-5-90 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
	node1: 172.17.5.90
	node2: 172.17.5.91
	node3: 172.17.5.92
	vip-master: 172.17.5.99
	vip-slave:  172.17.5.98

## 2.配置Linux集群环境 
### (1) 安装Pacemaker和Corosync包

在所有节点执行：

	$ sudo yum install -y pacemaker pcs psmisc policycoreutils-python postgresql-server

### (2) 禁用防火墙 

在所有节点执行：

	$ sudo setenforce 0
	$ sudo sed -i.bak "s/SELINUX=enforcing/SELINUX=permissive/g" /etc/selinux/config
	$ sudo systemctl disable firewalld.service
	$ sudo systemctl stop firewalld.service
	$ sudo iptables --flush

### (3) 启用pcs

在所有节点执行：

	$ sudo systemctl start pcsd.service
	$ sudo systemctl enable pcsd.service
	ln -s '/usr/lib/systemd/system/pcsd.service' '/etc/systemd/system/multi-user.target.wants/pcsd.service'
	$ echo hacluster | sudo passwd hacluster --stdin
	Changing password for user hacluster.
	Changing password for user hacluster.
	passwd: all authentication tokens updated successfully.

### (4) 集群认证 

在任何一个节点上执行，这里选择node1： 

	$ sudo pcs cluster auth -u hacluster -p hacluster 172.17.5.90 172.17.5.91 172.17.5.92
	172.17.5.90: Authorized
	172.17.5.91: Authorized
	172.17.5.92: Authorized

### (5) 同步配置 

在node1上执行： 

	$ sudo pcs cluster setup --last_man_standing=1 --name pgcluster 172.17.5.90 172.17.5.91 172.17.5.92
	Shutting down pacemaker/corosync services...
	Redirecting to /bin/systemctl stop  pacemaker.service
	Redirecting to /bin/systemctl stop  corosync.service
	Killing any remaining services...
	Removing all cluster configuration files...
	172.17.5.90: Succeeded
	172.17.5.91: Succeeded
	172.17.5.92: Succeeded

### (6) 启动集群 

在node1上执行： 

	$ sudo pcs cluster start --all
	172.17.5.90: Starting Cluster...
	172.17.5.91: Starting Cluster...
	172.17.5.92: Starting Cluster...

### (7) 检验 

1）检验corosync 

在node1上执行： 

	$ sudo pcs status corosync
	Membership information
	----------------------
	    Nodeid      Votes Name
	         1          1 172.17.5.90 (local)
	         2          1 172.17.5.91
	         3          1 172.17.5.92

2）检验pacemaker 

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: no stonith devices and stonith-enabled is not false
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Mon Oct 19 15:08:06 2015          Last change:
	Stack: unknown
	Current DC: NONE
	0 nodes and 0 resources configured
	Full list of resources:
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/disabled

## 3.安装和配置PostgreSQL 
### (1) 创建目录 

在所有节点上执行：

	$ sudo mkdir -p /data/postgresql/{data,xlog_archive}
	$ sudo chown -R postgres:postgres /data/postgresql/
	$ sudo chmod 0700 /data/postgresql/data

### (2) 初始化db 

在node1上执行： 

	$ sudo su - postgres
	$ initdb -D /data/postgresql/data/
	The files belonging to this database system will be owned by user "postgres".
	This user must also own the server process.
	The database cluster will be initialized with locale "en_US.UTF-8".
	The default database encoding has accordingly been set to "UTF8".
	The default text search configuration will be set to "english".
	fixing permissions on existing directory /data/postgresql/data ... ok
	creating subdirectories ... ok
	selecting default max_connections ... 100
	selecting default shared_buffers ... 32MB
	creating configuration files ... ok
	creating template1 database in /data/postgresql/data/base/1 ... ok
	initializing pg_authid ... ok
	initializing dependencies ... ok
	creating system views ... ok
	loading system objects' descriptions ... ok
	creating collations ... ok
	creating conversions ... ok
	creating dictionaries ... ok
	setting privileges on built-in objects ... ok
	creating information schema ... ok
	loading PL/pgSQL server-side language ... ok
	vacuuming database template1 ... ok
	copying template1 to template0 ... ok
	copying template1 to postgres ... ok
	WARNING: enabling "trust" authentication for local connections
	You can change this by editing pg_hba.conf or using the option -A, or
	--auth-local and --auth-host, the next time you run initdb.
	Success. You can now start the database server using:
	    postgres -D /data/postgresql/data
	or
	    pg_ctl -D /data/postgresql/data -l logfile start

### (3) 修改配置文件 

在node1上执行： 

	$ vim /data/postgresql/data/postgresql.conf
	listen_addresses = '*'
	wal_level = hot_standby
	synchronous_commit = on
	archive_mode = on
	archive_command = 'cp %p /data/postgresql/xlog_archive/%f'
	max_wal_senders=5
	wal_keep_segments = 32
	hot_standby = on
	restart_after_crash = off
	replication_timeout = 5000
	wal_receiver_status_interval = 2
	max_standby_streaming_delay = -1
	max_standby_archive_delay = -1
	synchronous_commit = on
	restart_after_crash = off
	hot_standby_feedback = on
	$ vim /data/postgresql/data/pg_hba.conf
	local   all                 all                              trust
	host    all                 all     172.17.0.0/16            md5
	host    replication         all     172.17.0.0/16            md5

### (4) 启动 

在node1上执行： 

	$ pg_ctl -D /data/postgresql/data/ start
	server starting
	[    2015-10-16 08:51:31.451 UTC 53158 5620ba93.cfa6 1 0]LOG:  redirecting log output to logging collector process
	[    2015-10-16 08:51:31.451 UTC 53158 5620ba93.cfa6 2 0]HINT:  Future log output will appear in directory "pg_log".
	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# create role replicator with login replication password '8d5e9531-3817-460d-a851-659d2e51ca99';
	CREATE ROLE
	postgres=# \q

### (5) 制作slave 

在node2和node3上执行： 

	$ sudo su - postgres
	$ pg_basebackup -h 172.17.5.90 -U postgres -D /data/postgresql/data/ -X stream -P
	could not change directory to "/home/wenhang.pan"
	20127/20127 kB (100%), 1/1 tablespace
	node2:
	$ vim /data/postgresql/data/recovery.conf
	standby_mode = 'on'
	primary_conninfo = 'host=172.17.5.90 port=5432 user=replicator password=8d5e9531-3817-460d-a851-659d2e51ca99 application_name=node2'
	restore_command = ''
	recovery_target_timeline = 'latest'
	node3:
	$ vim /data/postgresql/data/recovery.conf
	standby_mode = 'on'
	primary_conninfo = 'host=172.17.5.90 port=5432 user=replicator password=8d5e9531-3817-460d-a851-659d2e51ca99 application_name=node3'
	restore_command = ''
	recovery_target_timeline = 'latest'

### (6) 启动slave 

在node2和node3上执行： 

	$ pg_ctl -D /data/postgresql/data/ start
	pg_ctl: another server might be running; trying to start server anyway
	server starting
	-bash-4.2$ LOG:  database system was interrupted while in recovery at log time 2015-10-16 08:19:07 GMT
	HINT:  If this has occurred more than once some data might be corrupted and you might need to choose an earlier recovery target.
	LOG:  entering standby mode
	LOG:  redo starts at 0/3000020
	LOG:  consistent recovery state reached at 0/30000E0
	LOG:  database system is ready to accept read only connections
	LOG:  streaming replication successfully connected to primary

### (7) 查看集群状态 

在node1上执行： 

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid |  usename   | application_name | client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state
	-------+----------+------------+------------------+--------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 10745 |    16384 | postgres   | node2            | 172.17.5.91 |                 |       43013 | 2015-10-16 02:54:02.279384+00 |         1911 | streaming | 39/7B000060   | 39/7B000060    | 39/7B000060    | 39/7B000000     |             0 | async
	 50361 |    16384 | postgres   | node3            | 172.17.5.92 |                 |       52073 | 2015-10-15 10:13:15.436745+00 |         1911 | streaming | 39/7B000060   | 39/7B000060    | 39/7B000060    | 39/7B000000     |             0 | async
	(2 rows)
	postgres=# \q

### (8) 停止PostgreSQL服务
在node1、node2和node3上执行：

	$ pg_ctl -D /data/postgresql/data/ -mi stop
	waiting for server to shut down.... done
	server stopped

## 4.配置自动切换 
### (1) 配置

在node1执行：

1）将配置步骤先写到脚本

	$ vim cluster_setup.sh
	# 将cib配置保存到文件
	pcs cluster cib pgsql_cfg                                                                   
	# 在pacemaker级别忽略quorum
	pcs -f pgsql_cfg property set no-quorum-policy="ignore"        
	# 禁用STONITH           
	pcs -f pgsql_cfg property set stonith-enabled="false"                    
	# 设置资源粘性，防止节点在故障恢复后发生迁移     
	pcs -f pgsql_cfg resource defaults resource-stickiness="INFINITY"       
	# 设置多少次失败后迁移
	pcs -f pgsql_cfg resource defaults migration-threshold="3"                 
	# 设置master节点虚ip
	pcs -f pgsql_cfg resource create vip-master IPaddr2 ip="172.17.5.99" cidr_netmask="24"    op start   timeout="60s" interval="0s"  on-fail="restart"    op monitor timeout="60s" interval="10s" on-fail="restart"    op stop    timeout="60s" interval="0s"  on-fail="block"                             
	# 设置slave节点虚ip                       
	pcs -f pgsql_cfg resource create vip-slave IPaddr2 ip="172.17.5.98" cidr_netmask="24"    op start   timeout="60s" interval="0s"  on-fail="restart"    op monitor timeout="60s" interval="10s" on-fail="restart"    op stop    timeout="60s" interval="0s"  on-fail="block"                                                        
	# 设置pgsql集群资源
	# pgctl、psql、pgdata和config等配置根据自己的环境修改
	pcs -f pgsql_cfg resource create pgsql pgsql pgctl="/opt/pgsql/bin/pg_ctl" psql="/opt/pgsql/bin/psql" pgdata="/data/postgresql/data/" config="/data/postgresql/data/postgresql.conf" rep_mode="sync" node_list="zhaopin-5-90 zhaopin-5-91 zhaopin-5-92" master_ip="172.17.5.98"  repuser="replicator" primary_conninfo_opt="password=8d5e9531-3817-460d-a851-659d2e51ca99 keepalives_idle=60 keepalives_interval=5 keepalives_count=5" restore_command="cp /data/postgresql/xlog_archive/%f %p" restart_on_promote='true' op start   timeout="60s" interval="0s"  on-fail="restart" op monitor timeout="60s" interval="4s" on-fail="restart" op monitor timeout="60s" interval="3s"  on-fail="restart" role="Master" op promote timeout="60s" interval="0s"  on-fail="restart" op demote  timeout="60s" interval="0s"  on-fail="stop" op stop    timeout="60s" interval="0s"  on-fail="block"       
	 # 设置master/slave模式
	pcs -f pgsql_cfg resource master pgsql-cluster pgsql master-max=1 master-node-max=1 clone-max=3 clone-node-max=1 notify=true                                                                       
	# 配置master ip组
	pcs -f pgsql_cfg resource group add master-group vip-master        
	# 配置slave ip组     
	pcs -f pgsql_cfg resource group add slave-group vip-slave                 
	# 配置master ip组绑定master节点
	pcs -f pgsql_cfg constraint colocation add master-group with master pgsql-cluster INFINITY    
	# 配置启动master节点
	pcs -f pgsql_cfg constraint order promote pgsql-cluster then start master-group symmetrical=false score=INFINITY                                 
	# 配置停止master节点                                                                   
	pcs -f pgsql_cfg constraint order demote  pgsql-cluster then stop  master-group symmetrical=false score=0                                                                                                                
	# 配置slave ip组绑定slave节点
	pcs -f pgsql_cfg constraint colocation add slave-group with slave pgsql-cluster INFINITY         
	# 配置启动slave节点
	pcs -f pgsql_cfg constraint order promote pgsql-cluster then start slave-group symmetrical=false score=INFINITY                               
	# 配置停止slave节点                                                                         
	pcs -f pgsql_cfg constraint order demote  pgsql-cluster then stop  slave-group symmetrical=false score=0                                                                                                                  
	# 把配置文件push到cib
	pcs cluster cib-push pgsql_cfg

2）执行操作文件

	$ sudo sh cluster_setup.sh

### (2) 查看状态 

1）查看cluster状态 

在node1上执行： 

	$ sudo pcs status
	Cluster name: pgcluster
	WARNING: corosync and pacemaker node names do not match (IPs used in setup?)
	Last updated: Mon Oct 19 15:10:52 2015          Last change: Mon Oct 19 15:10:12 2015 by root via crm_attribute on zhaopin-5-92
	Stack: corosync
	Current DC: zhaopin-5-90 (version 1.1.13-a14efad) - partition with quorum
	3 nodes and 5 resources configured
	Online: [ zhaopin-5-90 zhaopin-5-91 zhaopin-5-92 ]
	Full list of resources:
	 Master/Slave Set: pgsql-cluster [pgsql]
	     Masters: [ zhaopin-5-92 ]
	     Slaves: [ zhaopin-5-90 zhaopin-5-91 ]
	 Resource Group: master-group
	     vip-master (ocf::heartbeat:IPaddr2):       Started zhaopin-5-92
	 Resource Group: slave-group
	     vip-slave  (ocf::heartbeat:IPaddr2):       Started zhaopin-5-90
	PCSD Status:
	  zhaopin-5-90 (172.17.5.90): Online
	  zhaopin-5-91 (172.17.5.91): Online
	  zhaopin-5-92 (172.17.5.92): Online
	Daemon Status:
	  corosync: active/disabled
	  pacemaker: active/disabled
	  pcsd: active/disabled

2）查看PostgreSQL集群状态

在node3上执行： 

	$ psql -U postgres
	psql (9.2.13)
	Type "help" for help.
	postgres=# select * from pg_stat_replication ;
	  pid  | usesysid |  usename   | application_name |  client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state
	-------+----------+------------+------------------+---------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
	 11522 |    16384 | replicator | zhaopin-5-91     | 172.17.5.91   |                 |       41356 | 2015-10-19 07:10:01.898257+00 |         1915 | streaming | 81/D9000000   | 81/D9000000    | 81/D9000000    | 81/D9000000     |             2 | potential
	 11532 |    16384 | replicator | zhaopin-5-90     | 172.17.5.99   |                 |       41786 | 2015-10-19 07:10:01.945532+00 |         1915 | streaming | 81/D9000000   | 81/D9000000    | 81/D9000000    | 81/D9000000     |             1 | sync
	(2 rows)

## 5.参考

[从头开始搭建集群](http://clusterlabs.org/doc/zh-CN/Pacemaker/1.1-pcs/html-single/Clusters_from_Scratch/index.html#_verify_corosync_installation)

[PgSQL Replicated Cluster](http://clusterlabs.org/wiki/PgSQL_Replicated_Cluster)
