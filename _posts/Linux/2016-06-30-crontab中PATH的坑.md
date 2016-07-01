---
layout: post
title: crontab中PATH的坑
categories: Linux
---

<!--more-->

## 1. 现象

在命令行里面能正常运行脚本，在crontab里面报错：

lsof: No such file or directory

看报错信息， 应该是找不到lsof这个命令

## 2. 排查

查看lsof命令的路径：

    $ type -a lsof
    lsof is /usr/bin/lsof

查看系统的PATH路径：

    $ echo $PATH
    /usr/lib64/ccache:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/opt/dell/srvadmin/bin:/bin

里面是包含/usr/sbin的，这能解释为什么命令行运行正常了，那就是crontab的PATH的问题了

在crontab运行一个echo $PATH的脚本，结果如下

    /usr/bin:/bin

果真是crontab的原因

## 3. 原因

    $ man 5 crontab

在man page里面找到

Several  environment variables are set up automatically by the cron(8) daemon.  SHELL is set to /bin/sh, and LOGNAME and HOME are set from the /etc/passwd line of the crontab's owner. PATH is set to "/usr/bin:/bin".  HOME, SHELL, and PATH may be overridden by settings in the crontab; LOGNAME is the user that the job is running from, and may not be changed.

## 4. 解决办法

在脚本里面export PATH=/usr/sbin:$PATH, 或者把crontab改成

    * * * * * source /etc/profile; your command

