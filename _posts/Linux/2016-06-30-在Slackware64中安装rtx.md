---
layout: post
title: 在Slackware64中安装rtx
categories: Linux
---

<!--more-->

## 1. 安装Multilib of Alien
在slackware64下执行如下命令，下载和安装32位库，安装完后需要重启才能生效，因为修改了lib库：

    $ mkdir multlib
    $ lftp -c 'open http://slackware.com/~alien/multilib/; mirror 14.1'
    $ cd 14.1
    $ sudo upgradepkg --reinstall --install-new *.t?z
    $ sudo upgradepkg --install-new slackware64-compat32/*-compat32/*.t?z

## 2. 安装Wine

    $ wget http://sourceforge.net/projects/wine/files/Slackware%20Packages/1.7.8/x86_64/wine-1.7.8-x86_64-1sg.txz
    $ sudo installpkg wine-1.7.8-x86_64-1sg.txz

## 3. 安装Winetricks and Cabextract
**winetricks:**

    $ wget http://winetricks.org/winetricks
    $ chmod +x winetricks
    $ cp winetricks /usr/local/bin

**cabextract:**

    $ wget http://slackbuilds.org/slackbuilds/14.1/system/cabextract.tar.gz
    $ tar zxvf cabextract.tar.gz
    $ cd cabextract/
    $ wget http://www.cabextract.org.uk/cabextract-1.4.tar.gz
    $ ./cabextract.SlackBuild
    $ sudo installpkg /tmp/cabextract-1.4-x86_64-1_SBo.tgz

## 4. 配置Wine

    $ winecfg
    $ winetricks msxml3 gdiplus riched20 riched30 vcrun6 vcrun2005sp1

## 5. 安装RTX

    $ wget http://dldir1.qq.com/foxmail/rtx/rtxclient2013formal.exe
    $ wine rtxclient2012formal.exe

## 6. 乱码解决

 把windows系统C:\Windows\Fonts\下的字体全部复制到linux的/usr/share/fonts/windows(这个目录需要自己创建)目录下，rtx的菜单栏应该可以正常显示中文了；如果字体乱码，选择菜单栏的编辑字体，选择宋体，然后重启rtx就可以了。

如果还是乱码的话，把下面的内容保存为rtx.reg, 执行regedit rtx.reg，然后把字体simsun.ttf复制到~/.wine/drive_c/windows/Fonts目录下。 

    $ vim rtx.reg
        REGEDIT4
        [HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes]
        "Arial"="simsun"
        "Arial CE,238"="simsun"
        "Arial CYR,204"="simsun"
        "Arial Greek,161"="simsun"
        "Arial TUR,162"="simsun"
        "Courier New"="simsun"
        "Courier New CE,238"="simsun"
        "Courier New CYR,204"="simsun"
        "Courier New Greek,161"="simsun"
        "Courier New TUR,162"="simsun"
        "FixedSys"="simsun"
        "Helv"="simsun"
        "Helvetica"="simsun"
        "MS Sans Serif"="simsun"
        "MS Shell Dlg"="simsun"
        "MS Shell Dlg 2"="simsun"
        "System"="simsun"
        "Tahoma"="simsun"
        "Times"="simsun"
        "Times New Roman CE,238"="simsun"
        "Times New Roman CYR,204"="simsun"
        "Times New Roman Greek,161"="simsun"
        "Times New Roman TUR,162"="simsun"
        "Tms Rmn"="simsun"

    $ regedit rtx.reg
    $ cp simsun.ttf ~/.wine/drive_c/windows/Fonts
