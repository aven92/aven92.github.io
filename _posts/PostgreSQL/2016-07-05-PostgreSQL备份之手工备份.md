---
layout: post
title: PostgreSQL备份之手工备份
categories: PostgreSQL
---

<!--more-->

## 1. 备份
### (1) 需要保证 archive_mode = on 和 archive_command 是有效的
### (2) 在 master 节点上连接上数据库并执行

    SELECT pg_start_backup('label', true);

pg_start_backup第二个参数设置为true的好处是备份开始会执行一个检查点，设置为true能尽快的完成检查点，并且能减少备份期间产生的wal文件，但是会对查询有影响，默认设置的checkpoint_completion_target的一半的时间，半夜的备份建议打开

### (3) 把物理文件复制到别的地方，可进行压缩或者使用nc，rsync等其他工具传送到其他地方

排除掉一些文件：

1） pg_log/下的所有日志文件，也可以一起备份

2） pg_xlog/下的所有wal文件

3） pg_xlog/archive_status/*下的所有文件

4） recovery.conf如果在从库下备份需要排除

5） postmaster.pid， postmaster.opts

### (4) 复制完后，连接数据库执行

    SELECT pg_stop_backup();

## 2.恢复
### (1) 将备份转化（解压或者复制）到要恢复的数据目录
### (2) 准备recovery.conf

    recovery_target_time='2015-08-04 12:00:00+8' #需要恢复到的时间点
    pause_at_recovery_target=true                #到恢复目标后暂停
    recovery_target_inclusive=false              #到恢复目标后是否停止恢复，设置为true会把recovery.conf重命名为recovery.done
    restore_command='cp /data/xlog/%f %p'        #wal文件归档的位置，必须指定


不能设置standby_mode = ‘on’ ，recovery_target_time需要在基础备份之后，还可以恢复到指定事务id和指定备份位置，详见：http://www.postgresql.org/docs/9.4/static/recovery-target-settings.html#RECOVERY-TARGET-INCLUSIVE

### (3) 启动备份数据库

    pg_ctl -D /data/restore_data start

参考：http://www.postgresql.org/docs/9.4/interactive/continuous-archiving.html
