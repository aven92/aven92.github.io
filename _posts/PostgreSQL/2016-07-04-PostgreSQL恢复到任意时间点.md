---
layout: post
title: PostgreSQL恢复到任意时间点
categories: PostgreSQL
---

<!--more-->

参考: [Recovery Configuration](http://www.postgresql.org/docs/9.4/static/recovery-config.html)

## 1. 获取基础备份
  你可以使用 pg_basebackup 或者其他工具获取Postgres的基础备份， 还要有基础备份到恢复时间点之前的 xlog 文件

## 2. 准备
### recovery.conf

	recovery_target_time='2015-05-08 12:00:00+8'
	pause_at_recovery_target=true
	recovery_target_inclusive=false
	restore_command='cp /xlog/%f %p'

* 注意:
  你不能在 recovery.conf 中设置 standby_mode = 'on' , 并且 recovery_target_time 应该在基础备份的结束时间之后.

### 删除或者重命名 backup_label 

### 启动 Postgres 实例
  当 Postgres 实例到达你恢复的时间, recovery.conf 会变成 recovery.done.
