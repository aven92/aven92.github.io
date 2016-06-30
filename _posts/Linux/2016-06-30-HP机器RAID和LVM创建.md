---
layout: post
title: HP机器RAID和LVM创建
categories: Linux
---

<!--more-->

# 1.环境

	[root@localhost ~]# uname -a
	Linux localhost.localdomain 2.6.32-220.el6.x86_64 #1 SMP Tue Dec 6 19:48:22 GMT 2011 x86_64 x86_64 x86_64 GNU/Linux
	[root@localhost ~]# cat /etc/issue
	CentOS release 6.7 (Final)
	Kernel \r on an \m
	[root@localhost ~]# dmidecode -t bios
	# dmidecode 2.12
	SMBIOS 2.7 present.
	
	Handle 0x0000, DMI type 0, 24 bytes
	BIOS Information
	        Vendor: HP
	        Version: P71
	        Release Date: 03/01/2013
	        Address: 0xF0000
	        Runtime Size: 64 kB
	        ROM Size: 8192 kB

# 2.安装HP磁盘管理工具

	[root@localhost ~]# rpm -ivh http://mirror.nforce.com/pub/software/raidtools/hpacucli-tool/hpacucli-9.40-12.0.x86_64.rpm


# 3.配置RAID
## (1) 产看当前RAID状态

	[root@localhost ~]# hpacucli ctrl all show status

	Smart Array P420i in Slot 0 (Embedded)
	   Controller Status: OK
	   Cache Status: OK
	   Battery/Capacitor Status: OK
   
	[root@localhost ~]# hpacucli ctrl all show config

	Smart Array P420i in Slot 0 (Embedded)    (sn: 500143802626A660)

	   array A (SAS, Unused Space: 0  MB)


	      logicaldrive 1 (558.9 GB, RAID 1, OK)

	      physicaldrive 1I:1:1 (port 1I:box 1:bay 1, SAS, 600 GB, OK)
	      physicaldrive 1I:1:2 (port 1I:box 1:bay 2, SAS, 600 GB, OK)

	   unassigned

	      physicaldrive 1I:1:3 (port 1I:box 1:bay 3, SAS, 600 GB, OK)
	      physicaldrive 1I:1:4 (port 1I:box 1:bay 4, SAS, 600 GB, OK)
	      physicaldrive 2I:1:5 (port 2I:box 1:bay 5, SAS, 600 GB, OK)
	      physicaldrive 2I:1:6 (port 2I:box 1:bay 6, SAS, 600 GB, OK)

	   SEP (Vendor ID PMCSIERA, Model SRCv8x6G) 380 (WWID: 500143802626A66F)


发现有两块盘做了RAID 1, 剩下4块盘没有做RAID。。

# (2) 创建RAID

	[root@localhost ~]# hpacucli ctrl slot=0 create type=ld drives=1I:1:3,1I:1:4,2I:1:5,2I:1:6 raid=1+0
	[root@localhost ~]# hpacucli ctrl all show config

	Smart Array P420i in Slot 0 (Embedded)    (sn: 500143802626A660)

	   array A (SAS, Unused Space: 0  MB)


	      logicaldrive 1 (558.9 GB, RAID 1, OK)

	      physicaldrive 1I:1:1 (port 1I:box 1:bay 1, SAS, 600 GB, OK)
	      physicaldrive 1I:1:2 (port 1I:box 1:bay 2, SAS, 600 GB, OK)

	   array B (SAS, Unused Space: 0  MB)


	      logicaldrive 2 (1.1 TB, RAID 1+0, OK)

	      physicaldrive 1I:1:3 (port 1I:box 1:bay 3, SAS, 600 GB, OK)
	      physicaldrive 1I:1:4 (port 1I:box 1:bay 4, SAS, 600 GB, OK)
	      physicaldrive 2I:1:5 (port 2I:box 1:bay 5, SAS, 600 GB, OK)
	      physicaldrive 2I:1:6 (port 2I:box 1:bay 6, SAS, 600 GB, OK)

	   SEP (Vendor ID PMCSIERA, Model SRCv8x6G) 380 (WWID: 500143802626A66F)


为剩下的4块盘创建了一个RAID 1+0

# 4.配置LVM
## (1) 创建分区

	[root@localhost ~]# cfdisk /dev/sdb


## (2) 创建物理卷

	[root@localhost ~]# pvs
	  PV         VG       Fmt  Attr PSize   PFree
	  /dev/sda2  VolGroup lvm2 a--  558.39g    0
	[root@localhost ~]# pvcreate /dev/sdb1
	  Physical volume "/dev/sdb1" successfully created
	[root@localhost ~]# pvs
	  PV         VG       Fmt  Attr PSize   PFree
	  /dev/sda2  VolGroup lvm2 a--  558.39g    0
	  /dev/sdb1           lvm2 ---    1.09t 1.09t


## (3) 将创建的物理卷添加到卷组

	[root@localhost ~]# vgs
	  VG       #PV #LV #SN Attr   VSize   VFree
	  VolGroup   1   3   0 wz--n- 558.39g    0
	[root@localhost ~]# vgextend VolGroup /dev/sdb1
	  Volume group "VolGroup" successfully extended
	[root@localhost ~]# vgs
	  VG       #PV #LV #SN Attr   VSize VFree
	  VolGroup   2   3   0 wz--n- 1.64t 1.09t


## (4) 创建逻辑卷

	[root@localhost ~]# lvs
	  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  lv_home VolGroup -wi-ao---- 380.25g
	  lv_root VolGroup -wi-ao----  50.00g
	  lv_swap VolGroup -wi-ao---- 128.14g
	[root@localhost ~]# lvcreate -L 1T -n lv_data VolGroup
	  Logical volume "lv_data" created.
	[root@localhost ~]# lvs
	  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  lv_data VolGroup -wi-a-----   1.00t
	  lv_home VolGroup -wi-ao---- 380.25g
	  lv_root VolGroup -wi-ao----  50.00g
	  lv_swap VolGroup -wi-ao---- 128.14g


# 5.格式化使用
## (1) 格式化

	[root@localhost ~]#  mkfs.ext4 /dev/VolGroup/lv_data
	mke2fs 1.41.12 (17-May-2010)
	Filesystem label=
	OS type: Linux
	Block size=4096 (log=2)
	Fragment size=4096 (log=2)
	Stride=64 blocks, Stripe width=128 blocks
	67108864 inodes, 268435456 blocks
	13421772 blocks (5.00%) reserved for the super user
	First data block=0
	Maximum filesystem blocks=4294967296
	8192 block groups
	32768 blocks per group, 32768 fragments per group
	8192 inodes per group
	Superblock backups stored on blocks:
	        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
	        102400000, 214990848

	Writing inode tables: done
	Creating journal (32768 blocks): done
	Writing superblocks and filesystem accounting information: done

	This filesystem will be automatically checked every 28 mounts or
	180 days, whichever comes first.  Use tune2fs -c or -i to override.
	[root@localhost ~]#  e2fsck -f /dev/VolGroup/lv_data
	e2fsck 1.41.12 (17-May-2010)
	Pass 1: Checking inodes, blocks, and sizes
	Pass 2: Checking directory structure
	Pass 3: Checking directory connectivity
	Pass 4: Checking reference counts
	Pass 5: Checking group summary information
	/dev/VolGroup/lv_data: 11/67108864 files (0.0% non-contiguous), 4262937/268435456 blocks

## (2) 挂载

	[root@localhost ~]# df -h
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/mapper/VolGroup-lv_root
	                       50G  5.1G   42G  11% /
	tmpfs                  63G     0   63G   0% /dev/shm
	/dev/sda1             485M   67M  393M  15% /boot
	/dev/mapper/VolGroup-lv_home
	                      375G  192G  164G  55% /home
	[root@localhost ~]# mkdir -p /data
	[root@localhost ~]# mount -t ext4 /dev/VolGroup/lv_data /data
	[root@localhost ~]# df -h
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/mapper/VolGroup-lv_root
	                       50G  5.1G   42G  11% /
	tmpfs                  63G     0   63G   0% /dev/shm
	/dev/sda1             485M   67M  393M  15% /boot
	/dev/mapper/VolGroup-lv_home
	                      375G  192G  164G  55% /home
	/dev/mapper/VolGroup-lv_data
	                     1008G  200M  957G   1% /data


# 6.参考

http://www.datadisk.co.uk/html_docs/redhat/hpacucli.htm

