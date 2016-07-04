---
layout: post
title: PostgreSQL日志使用rsyslog
categories: PostgreSQL
---

<!--more-->

## 1. 服务端

### (1) 安装syslog包

    $ sudo yum install -y rsyslog

### (2) 创建目录

    sudo mkdir -p /export/postgreslog/

### (3) 配置

    $ sudo vim /etc/rsyslog.conf
        # Provides TCP syslog reception
        $ModLoad imtcp.so
        $InputTCPServerRun 514

        #### GLOBAL DIRECTIVES ####
        $MainMsgQueueSize 500000
        $MainMsgQueueDequeueBatchSize 128
        $MainMsgQueueDiscardMark 20000
        $MainMsgQueueHighWaterMark 16000
        $MainMsgQueueLowWaterMark 4000

        $EscapeControlCharactersOnReceive off

        # Use default timestamp format
        $ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
        # log format
        $template pgFormat,"%msg:R,ERE,2,DFLT:^ \[[0-9]+-[0-9]+\] (#011)?(.*)--end%\n"
        # logfile name
        $template PostgresLogFile,"/export/postgreslog/%hostname%_%programname%/%$year%-%$month%-%$day%/postgresql-%$year%-%$month%-%$day%-%$hour%.log"

        # collect log of local0
        local0.* ?PostgresLogFile;pgFormat

### (4) 启动

    $ sudo /etc/init.d/rsyslog restart

## 2. 客户端

### (1) 安装syslog包

    $ sudo yum install -y rsyslog

### (2) 配置rsyslog

    sudo vim /etc/rsyslog.conf
        EscapeControlCharactersOnReceive off 
        local0.* @@rsyslog_server:514

### (3) 启动rsyslog

    $ sudo /etc/init.d/rsyslog restart

### (4) 配置 postgresql.conf

    $ vim postgresql.conf
        log_destination = 'syslog'
        syslog_facility = 'LOCAL0'
        syslog_ident = 'pg-test'

### (5) reload配置

    # select pg_reload_conf();

