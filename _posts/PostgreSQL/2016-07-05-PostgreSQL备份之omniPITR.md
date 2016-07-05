---
layout: post
title: PostgreSQL备份之omniPITR
categories: PostgreSQL
---

<!--more-->

## 1.准备
### (1) 安装

omnipitr是用perl写的，直接下载下来就可以用了

    # git clone https://github.com/omniti-labs/omnipitr.git /opt/omnipitr/
    Initialized empty Git repository in /opt/omnipitr/.git/
    remote: Counting objects: 2627, done.
    remote: Total 2627 (delta 0), reused 0 (delta 0), pack-reused 2627
    Receiving objects: 100% (2627/2627), 742.71 KiB | 28 KiB/s, done.
    Resolving deltas: 100% (1287/1287), done.

### (2) 检查

    # /opt/omnipitr/bin/sanity-check.sh
    Checking:
    - /opt/omnipitr/bin
    - /opt/omnipitr/lib
    9 programs, 31 libraries.
    Tar version
    All checked, and looks ok.

### (3) 创建目录

    # mkdir -p /data/omnipitr/test_94/{log,state,tmp}
    # chown -R postgres:postgres /data/omnipitr

## 2.备份
### (1) 从master节点上备份

1）修改archive_command

    test ! -f /data/omnipitr/test_94/tmp/backup/%f && cp %p /data/omnipitr/test_94/tmp/backup/%f

如果已经将wal归档到其他地方，不需要备份wal文件，就不需要配置这个

2）备份到本地

    $ /opt/omnipitr/bin/omnipitr-backup-master --host=localhost \    #master节点ip
    --username=postgres \    #连接master节点的用户
    --port=5432 \    #master节点的端口
    --data-dir=/data/postgres/data/test_94/ \    #master节点的数据目录
    --dst-local gzip=/data/postgres/backup/test_94/ \    #本地备份目录
    --xlogs=/data/omnipitr/test_94/tmp/backup \    #用于归档xlog的目录，执行命令前不能存在，如果不需要备份xlog，替换成--skip-xlogs
    --temp-dir=/data/omnipitr/test_94/tmp/ \    #创建临时文件的目录，可以不指定，默认是/tmp
    --pid-file=/data/omnipitr/test_94/state/base_backup.pid \    #进程文件，保证同时只有一个备份在执行
    --log=/data/omnipitr/test_94/log/base_backup.log \    #日志输出文件
    --verbose    #打印详细信息

3）备份的详细日志信息

    2015-09-09 04:50:54.185227 -0400 : 9766 : omnipitr-backup-master : LOG : Called with parameters: --host=localhost --username=postgres --port=5432 --data-dir=/data/postgres/data/test_94/ --dst-local gzip=/data/postgres/backup/test_94/ --xlogs=/data/omnipitr/test_94/tmp/backup --temp-dir=/data/omnipitr/test_94/tmp/ --pid-file=/data/omnipitr/test_94/state/base_backup.pid --log=/data/omnipitr/test_94/log/base_backup.log --verbose
    2015-09-09 04:50:54.363777 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [SELECT w, pg_xlogfile_name(w) from (select pg_start_backup('omnipitr') as w ) as x] took: 0.175s
    2015-09-09 04:50:54.366482 -0400 : 9766 : omnipitr-backup-master : LOG : pg_start_backup('omnipitr') returned 1/3C000028|00000001000000010000003C.
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : Script to make tarballs:
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : mkfifo \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : nice gzip \-\-stdout \- < \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0 > \/data\/postgres\/backup\/test_94\/QA\-5\-45\-data\-2015\-09\-09\.tar\.gz &
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : nice tar cf \- \-\-exclude\=pg_log\/\* \-\-exclude\=pg_xlog\/0\* \-\-exclude\=pg_xlog\/archive_status\/\* \-\-exclude\=postmaster\.pid test_94 2> \/data\/omnipitr\/test_94\/tmp\/omnipitr\-backup\-master\/9766\/tar\.stderr > \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : wait
    2015-09-09 04:50:54.368554 -0400 : 9766 : omnipitr-backup-master : LOG : rm \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:51:14.415299 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Compressing $PGDATA] took: 20.048s
    2015-09-09 04:51:15.483531 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [SELECT pg_stop_backup()] took: 1.066s
    2015-09-09 04:51:15.486756 -0400 : 9766 : omnipitr-backup-master : LOG : pg_stop_backup('omnipitr') returned 1/3C000128.
    2015-09-09 04:51:15.487734 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Making data archive] took: 21.300s
    2015-09-09 04:51:15.489015 -0400 : 9766 : omnipitr-backup-master : LOG : File 00000001000000010000003C.00000028.backup arrived after 0 seconds.
    2015-09-09 04:51:15.490275 -0400 : 9766 : omnipitr-backup-master : LOG : File 00000001000000010000003C arrived after 0 seconds.
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : Script to make tarballs:
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : mkfifo \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : nice gzip \-\-stdout \- < \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0 > \/data\/postgres\/backup\/test_94\/QA\-5\-45\-xlog\-2015\-09\-09\.tar\.gz &
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : nice tar cf \- test_94 2> \/data\/omnipitr\/test_94\/tmp\/omnipitr\-backup\-master\/9766\/tar\.stderr > \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : wait
    2015-09-09 04:51:15.491816 -0400 : 9766 : omnipitr-backup-master : LOG : rm \/tmp\/CommandPiper\-9766\-Ldy5aL\/fifo\-0
    2015-09-09 04:51:15.936762 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Compressing xlogs] took: 0.445s
    2015-09-09 04:51:15.947223 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Making xlog archive] took: 0.458s
    2015-09-09 04:51:15.948752 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Delivering to all remote destinations] took: 0.000s
    2015-09-09 04:51:15.950304 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Delivering meta files] took: 0.001s
    2015-09-09 04:51:15.951272 -0400 : 9766 : omnipitr-backup-master : LOG : Timer [Whole backup procedure] took: 21.764s
    2015-09-09 04:51:15.952141 -0400 : 9766 : omnipitr-backup-master : LOG : All done.

4） 备份文件

    $ ll /data/postgres/backup/test_94/
    total 95620
    -rw-rw-r--. 1 postgres postgres 97851471 Sep  9 04:56 QA-5-45-data-2015-09-09.tar.gz
    -rw-rw-r--. 1 postgres postgres       93 Sep  9 04:56 QA-5-45-meta-2015-09-09.tar.gz
    -rw-rw-r--. 1 postgres postgres    54742 Sep  9 04:56 QA-5-45-xlog-2015-09-09.tar.gz

5） 备份到远端

    $ /opt/omnipitr/bin/omnipitr-backup-master --host=localhost \
    --username=postgres \
    --port=5432 \
    --data-dir=/data/postgres/data/test_94/ \
    --dst-direct gzip=root@172.17.5.46:/data/postgres/backup/test_94/ \    #使用ssh方式传送备份
    --skip-xlogs \    #不备份wal文件
    --temp-dir=/data/omnipitr/test_94/tmp/ \
    --pid-file=/data/omnipitr/test_94/state/base_backup.pid \
    --log=/data/omnipitr/test_94/log/base_backup.log \
    --verbose 
    root@172.17.5.46's password:

### (2) 从slave节点备份

1) 备份

    $ /opt/omnipitr/bin/omnipitr-backup-slave --host=172.17.5.45 \    #master节点ip
    --username=postgres \
    --port=5432 \
    --call-master \    #保证在master上执行SELECT pg_start_backup( '...'  );和SELECT pg_stop_backup()；，否则做出来的备份可能无效,对应的用户名必须是superuser
    --data-dir=/data/postgres/data/test_94/ \
    --dst-local gzip=/data/postgres/backup/test_94/ \
    --source=/data/postgres/data/test_94/pg_xlog \
    --temp-dir=/data/omnipitr/test_94/tmp/ \
    --pid-file=/data/omnipitr/test_94/state/base_backup.pid \
    --log=/data/omnipitr/test_94/log/base_backup.log \
    --verbose
    Password for user postgres: 
    Password for user postgres: 
    Password for user postgres:

2） 备份日志

    2015-09-09 06:16:55.621897 -0400 : 23447 : omnipitr-backup-slave : LOG : Called with parameters: --host=172.17.5.45 --username=postgres --port=5432 --call-master --data-dir=/data/postgres/data/test_94/ --dst-local gzip=/data/postgres/backup/test_94/ --skip-xlogs --temp-dir=/data/omnipitr/test_94/tmp/ --pid-file=/data/omnipitr/test_94/state/base_backup.pid --log=/data/omnipitr/test_94/log/base_backup.log --verbose
    2015-09-09 06:16:58.856673 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [SELECT w, pg_xlogfile_name(w) from ( as w  ) as x] took: 3.220s
    2015-09-09 06:16:58.864733 -0400 : 23447 : omnipitr-backup-slave : LOG : pg_start_backup('omnipitr', true) returned 1/62000028|000000010000000100000062.
    2015-09-09 06:17:01.862033 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [select pg_read_file( 'backup_label', 0, ( pg_stat_file( 'backup_label'  )  ).size  )] took: 2.996s
    2015-09-09 06:17:01.863971 -0400 : 23447 : omnipitr-backup-slave : LOG : Waiting for checkpoint (based on backup_label from master) - CHECKPOINT LOCATION: 1/62000060
    2015-09-09 06:18:37.008788 -0400 : 23447 : omnipitr-backup-slave : LOG : Checkpoint .
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : Script to make tarballs:
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : mkfifo \/tmp\/CommandPiper\-23447\-buP6d5\/fifo\-0
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : nice gzip \-\-stdout \- < \/tmp\/CommandPiper\-23447\-buP6d5\/fifo\-0 > \/data\/postgres\/backup\/test_94\/node2\-data\-2015\-09\-09\.tar\.gz &
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : nice tar cf \- \-\-exclude\=test_94\/pg_log\/\* \-\-exclude\=test_94\/pg_xlog\/0\* \-\-exclude\=test_94\/pg_xlog\/archive_status\/\* \-\-exclude\=test_94\/recovery\.conf \-\-exclude\=test_94\/postmaster\.pid \-\-transform\=s\#\^data\/omnipitr\/test_94\/tmp\/omnipitr\-backup\-slave\/23447\/\#test_94\/\# test_94 \/data\/omnipitr\/test_94\/tmp\/omnipitr\-backup\-slave\/23447\/backup_label 2> \/data\/omnipitr\/test_94\/tmp\/omnipitr\-backup\-slave\/23447\/tar\.stderr > \/tmp\/CommandPiper\-23447\-buP6d5\/fifo\-0
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : wait
    2015-09-09 06:18:37.013235 -0400 : 23447 : omnipitr-backup-slave : LOG : rm \/tmp\/CommandPiper\-23447\-buP6d5\/fifo\-0
    2015-09-09 06:18:57.288130 -0400 : 23447 : omnipitr-backup-slave : LOG : tar stderr:
    2015-09-09 06:18:57.290137 -0400 : 23447 : omnipitr-backup-slave : LOG : ==============================================
    2015-09-09 06:18:57.290849 -0400 : 23447 : omnipitr-backup-slave : LOG : tar: Removing leading `/' from member names
    2015-09-09 06:18:57.291659 -0400 : 23447 : omnipitr-backup-slave : LOG : ==============================================
    2015-09-09 06:18:57.292502 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Compressing $PGDATA] took: 20.280s
    2015-09-09 06:19:34.578726 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [SELECT pg_stop_backup()] took: 37.285s
    2015-09-09 06:19:34.580554 -0400 : 23447 : omnipitr-backup-slave : LOG : pg_stop_backup() returned 1/62000128.
    2015-09-09 06:19:34.581542 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Making data archive] took: 158.950s
    2015-09-09 06:19:34.582615 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Making xlog archive] took: 0.000s
    2015-09-09 06:19:34.583894 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Delivering to all remote destinations] took: 0.000s
    2015-09-09 06:19:34.585621 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Delivering meta files] took: 0.000s
    2015-09-09 06:19:34.586895 -0400 : 23447 : omnipitr-backup-slave : LOG : Timer [Whole backup procedure] took: 158.955s
    2015-09-09 06:19:34.587940 -0400 : 23447 : omnipitr-backup-slave : LOG : All done.

3） 备份文件

    # ll /data/postgres/backup/test_94/
    total 95548
    -rw-r--r--. 1 postgres postgres 97836713 Sep  9 06:18 node2-data-2015-09-09.tar.gz
    -rw-r--r--. 1 postgres postgres       91 Sep  9 06:19 node2-meta-2015-09-09.tar.gz

4) 备注

默认执行pg_start_backup()是没有加true的，上面红色的部分是我修改了下代码，这样执行checkpoint会快很多

