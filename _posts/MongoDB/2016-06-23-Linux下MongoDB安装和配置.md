---
layout: post
title: Linux 下 MongoDB 安装和配置
categories: MongoDB
---

<!--more-->

# 1. 环境

    $ cat /etc/redhat-release 
    CentOS Linux release 7.0.1406 (Core) 
    $ uname -a
    Linux zhaopin-2-201 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux

# 2. 准备
## (1) 下载包

    $ sudo wget -c -P /opt/ https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.0.6.tgz    
    --2015-09-23 10:18:24--  https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.0.6.tgz
    Resolving fastdl.mongodb.org (fastdl.mongodb.org)... 54.230.108.68, 54.230.108.110, 54.230.108.116, ... 
    Connecting to fastdl.mongodb.org (fastdl.mongodb.org)|54.230.108.68|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 50210575 (48M) [application/x-gzip]
    Saving to: ‘/opt/mongodb-linux-x86_64-rhel70-3.0.6.tgz’
    100%[======================================================================================================================================================================================================================>] 50,210,575  1.20MB/s   in 50s     
    2015-09-23 10:19:19 (978 KB/s) - ‘/opt/mongodb-linux-x86_64-rhel70-3.0.6.tgz’ saved [50210575/50210575]

## (2) 环境

    $ cd /opt/
    $ sudo tar -zxf mongodb-linux-x86_64-rhel70-3.0.6.tgz
    $ ll
    total 49052
    drwxr-xr-x. 3 root root     4096 Sep 23 10:20 mongodb-linux-x86_64-rhel70-3.0.6
    -rw-r--r--. 1 root root 50210575 Aug 24 11:38 mongodb-linux-x86_64-rhel70-3.0.6.tgz
    drwxr-xr-x. 8 root root     4096 Sep 21 15:41 omnipitr
    drwxr-xr-x. 6 root root     4096 Sep 21 15:53 pg94
    lrwxrwxrwx. 1 root root        9 Sep 21 15:53 pgsql -> /opt/pg94
    drwxr-xr-x. 2 root root     4096 Jun 10  2014 rh
    $ sudo ln -sf /opt/mongodb-linux-x86_64-rhel70-3.0.6 /opt/mongodb
    $ ll
    total 49052
    lrwxrwxrwx. 1 root root       38 Sep 23 10:21 mongodb -> /opt/mongodb-linux-x86_64-rhel70-3.0.6
    drwxr-xr-x. 3 root root     4096 Sep 23 10:20 mongodb-linux-x86_64-rhel70-3.0.6
    -rw-r--r--. 1 root root 50210575 Aug 24 11:38 mongodb-linux-x86_64-rhel70-3.0.6.tgz
    drwxr-xr-x. 8 root root     4096 Sep 21 15:41 omnipitr
    drwxr-xr-x. 6 root root     4096 Sep 21 15:53 pg94
    lrwxrwxrwx. 1 root root        9 Sep 21 15:53 pgsql -> /opt/pg94
    drwxr-xr-x. 2 root root     4096 Jun 10  2014 rh
    $
    $ sudo vim /etc/profile.d/mongodb.sh
    #!/bin/env bash
    export PATH="/opt/mongodb/bin:$PATH"
    $ source /etc/profile.d/mongodb.sh
    $ mongo --version
    MongoDB shell version: 3.0.6

# 3. 配置

    $ sudo mkdir -p /data/mongodb/db0/{data,backup,log,conf}
    $ sudo vim /data/mongodb/db0/conf/mongodb.conf
    # base
    port = 27017
    maxConns = 800
    filePermissions = 0700
    fork = true
    noauth = true
    directoryperdb = true
    dbpath = /data/mongodb/db0/data
    pidfilepath = /data/mongodb/db0/data/mongodb.pid
    journal = true

    # security
    nohttpinterface = true
    rest = false

    # log
    logpath = /data/mongodb/db0/log/mongodb.log
    logRotate = rename
    logappend = true
    slowms = 50
    profile = 0
    verbose = false
    objcheck = false

# 4. 启动

    $ sudo systemctl stop firewalld.service
    $ sudo systemctl disable firewalld.service
    rm '/etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service'
    rm '/etc/systemd/system/basic.target.wants/firewalld.service'
    $ sudo /opt/mongodb/bin/mongod --config /data/mongodb/db0/conf/mongodb.conf
    $ mongo
    MongoDB shell version: 3.0.6
    connecting to: test
    > db.version();
    3.0.6
    >
    bye
