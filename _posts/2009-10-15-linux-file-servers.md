---
layout: post
title:  "Linux配置DNS服务器"
date:   2009-10-14 +0800
categories: Linux
tags: Linux
author: dbtan
---

* content
{:toc}


文件服务器：`samba`、`ftp`、`nfs`






### 一、samba服务器：

samba：提供与windows“网上邻居”的相互通信。

端口：137、138、139。

安装包：（三个包）

```
[root@vm51: /etc/samba]#rpm -qf smb.conf
samba-common-3.0.23c-2
[root@vm51: /etc/samba]#rpm -qa | grep samba
samba-client-3.0.23c-2
samba-3.0.23c-2
system-config-samba-1.2.39-1.el5
samba-common-3.0.23c-2
```

![clip_image001](https://o8foyu42q.qnssl.com/tq_notes/linux-file-servers/clip_image001.gif)

服务器端：

主控配置文件：`/etc/samba/smb.conf`

分两大段：① global 全局属性；② share definitions 共享定义。

```
[root@vm51: /etc/samba]#vim smb.conf
# This is the main Samba configuration file. You should read the
# smb.conf(5) manual page in order to understand the options listed
# here. Samba has a huge number of configurable options (perhaps too
# many!) most of which are not shown in this example
#
# For a step to step guide on installing, configuring and using samba,
# read the Samba-HOWTO-Collection. This may be obtained from:
# http://www.samba.org/samba/docs/Samba-HOWTO-Collection.pdf
#
# Many working examples of smb.conf files can be found in the
# Samba-Guide which is generated daily and can be downloaded from:
# http://www.samba.org/samba/docs/Samba-Guide.pdf
#
# Any line which starts with a ; (semi-colon) or a # (hash)
# is a comment and is ignored. In this example we will use a #
# for commentry and a ; for parts of the config file that you
# may wish to enable
#
# NOTE: Whenever you modify this file you should run the command "testparm"
# to check that you have not made any basic syntactic errors.
#
#=================== Global Settings ==================================
[global]
# workgroup = NT-Domain-Name or Workgroup-Name, eg: MIDEARTH
workgroup = MYGROUP 广播域的最小范围。
# server string is the equivalent of the NT Description field
server string = Samba Server%v 说明信息。%v：版本号
# Security mode. Defines in which mode Samba will operate. Possible
# values are share, user, server, domain and ads. Most people will want
# user level security. See the Samba-HOWTO-Collection for details.
security = user =user：要通过用户身份验证。
＝share:不用任何身份验证。
＝server:验证交给一台验证服务器，域（共享）管理。
＝domain：验证交给域控制器（DC）
# This option is important for security. It allows you to restrict
# connections to machines which are on your local network. The
# following example restricts access to two C class networks and
# the "loopback" interface. For more examples of the syntax see
# the smb.conf man page
; hosts allow = 192.168.1. 192.168.2. 127. 监听网段
# If you want to automatically load your printer list rather
# than setting them up individually then you'll need this
load printers = yes 自动加载打印机
注：smb不支持打印服务，只是共享打印机。
# you may wish to override the location of the printcap file
; printcap name = /etc/printcap
# on SystemV system setting printcap name to lpstat should allow
# you to automatically obtain a printer list from the SystemV spool
# system
; printcap name = lpstat
# It should not be necessary to specify the print system type unless
# it is non-standard. Currently supported print systems include:
# bsd, cups, sysv, plp, lprng, aix, hpux, qnx
; printing = cups
# This option tells cups that the data has already been rasterized
cups options = raw 打印机设置属性
# Uncomment this if you want a guest account, you must add this to /etc/passwd
# otherwise the user "nobody" is used
; guest account = pcguest
# this tells Samba to use a separate log file for each machine
# that connects
log file = /var/log/samba/%m.log ％m：客户端主机名，实现每个用户都有一个.log日志
# Put a capping on the size of the log files (in Kb).
max log size = 50 以日志大小限制，50K
# Use password server option only with security = server
# The argument list may include:
# password server = My_PDC_Name [My_BDC_Name] [My_Next_BDC_Name]
# or to auto-locate the domain controller/s
# password server = *
; password server = <NT-Server-Name>
# Use the realm option only with security = ads
# Specifies the Active Directory realm the host is part of
; realm = MY_REALM
# Backend to store user information in. New installations should
# use either tdbsam or ldapsam. smbpasswd is available for backwards
# compatibility. tdbsam requires no further configuration.
; passdb backend = tdbsam 密码备份
# Using the following line enables you to customise your configuration
# on a per machine basis. The %m gets replaced with the netbios name
# of the machine that is connecting.
# Note: Consider carefully the location in the configuration file of
# this line. The included file is read at that point.
; include = /usr/local/samba/lib/smb.conf.%m
# Configure Samba to use multiple interfaces
# If you have multiple network interfaces then you must list them
# here. See the man page for details.
; interfaces = 192.168.12.2/24 192.168.13.2/24 监听网卡（或网段）
# Browser Control Options:
# set local master to no if you don't want Samba to become a master
# browser on your network. Otherwise the normal election rules apply
; local master = no
注：工作组中随机选出一台机器为域浏览器功能，不“即时更新”。
# OS Level determines the precedence of this server in master browser
# elections. The default value should be reasonable
; os level = 33 权重：高0～65低
# Domain Master specifies Samba to be the Domain Master Browser. This
# allows Samba to collate browse lists between subnets. Don't use this
# if you already have a Windows NT domain controller doing this job
; domain master = yes
# Preferred Master causes Samba to force a local browser election on startup
# and gives it a slightly higher chance of winning the election
; preferred master = yes
#域控Enable this if you want Samba to be a domain logon server for
# Windows95 workstations.
; domain logons = yes
# if you enable domain logons then you may want a per-machine or
# per user logon script
# run a specific logon batch file per workstation (machine)
; logon script = %m.bat 客户端主机名
# run a specific logon batch file per username
; logon script = %U.bat 服务端用户名
# Where to store roving profiles (only for Win95 and WinNT)
# %L substitutes for this servers netbios name, %U is username
# You must uncomment the [Profiles] share below
; logon path = \\%L\Profiles\%U 登录路径
# Name Resolution 解析Windows Internet Name Serving Support Section:
# WINS Support - Tells the NMBD component of Samba to enable it's WINS Server
; wins support = yes
# WINS Server - Tells the NMBD components of Samba to be a WINS Client
# Note: Samba can be either a WINS Server, or a WINS Client, but NOT both
; wins server = w.x.y.z
# WINS Proxy - Tells Samba to answer name resolution queries on
# behalf of a non WINS capable client, for this to work there must be
# at least one WINS Server on the network. The default is NO.
; wins proxy = yes
# DNS Proxy - tells Samba whether or not to try to resolve NetBIOS names
# via DNS nslookups. The default is NO.
dns proxy = no
# These scripts are used on a domain controller or stand-alone
# machine to add or delete corresponding unix accounts
; add user script = /usr/sbin/useradd %u
; add group script = /usr/sbin/groupadd %g
; add machine script = /usr/sbin/adduser -n -g machines -c Machine -d /dev/null -s /bin/false %u
; delete user script = /usr/sbin/userdel %u
; delete user from group script = /usr/sbin/deluser %u %g
; delete group script = /usr/sbin/groupdel %g
#========================= Share Definitions ==============================
[homes]
comment = Home Directories
browseable = no
writable = yes
# Un-comment the following and create the netlogon directory for Domain Logons
; [netlogon]
; comment = Network Logon Service
; path = /usr/local/samba/lib/netlogon
; guest ok = yes
; writable = no
; share modes = no
# Un-comment the following to provide a specific roving profile share
# the default is to use the user's home directory
;[Profiles]
; path = /usr/local/samba/profiles
; browseable = no
; guest ok = yes
# NOTE: If you have a BSD-style print system there is no need to
# specifically define each individual printer
[printers]
comment = All Printers 注释
path = /usr/spool/samba
browseable = no
# Set public = yes to allow user 'guest account' to print
guest ok = no 匿名用户是否可以使用
writable = no 用户是否可写
printable = yes
# This one is useful for people to share files
;[tmp]
; comment = Temporary file space
; path = /tmp
; read only = no
; public = yes
# A publicly accessible directory, but read only, except for people in
# the "staff" group
;[public]
; comment = Public Stuff
; path = /home/samba
; public = yes
; writable = yes
; printable = no
; write list = @staff
# Other examples.
#
# A private printer, usable only by fred. Spool data will be placed in fred's
# home directory. Note that fred must have write access to the spool directory,
# wherever it is.
;[fredsprn]
; comment = Fred's Printer
; valid users = fred 只有fred用户可用
; path = /homes/fred
; printer = freds_printer
; public = no
; writable = no
; printable = yes
# A private directory, usable only by fred. Note that fred requires write
# access to the directory.
;[fredsdir]
; comment = Fred's Service
; path = /usr/somewhere/private
; valid users = fred
; public = no
; writable = yes
; printable = no
# a service which has a different directory for each machine that connects
# this allows you to tailor configurations to incoming machines. You could
# also use the %U option to tailor it by user name.
# The %m gets replaced with the machine name that is connecting.
;[pchome]
; comment = PC Directories
; path = /usr/pc/%m
; public = no
; writable = yes
# A publicly accessible directory, read/write to all users. Note that all files
# created in the directory by users will be owned by the default user, so
# any user with access can delete any other user's files. Obviously this
# directory must be writable by the default user. Another user could of course
# be specified, in which case all files would be owned by that user instead.
;[public]
; path = /usr/somewhere/else/public
; public = yes
; only guest = yes
; writable = yes
; printable = no
# The following two entries demonstrate how to share a directory so that two
# users can place files there that will be owned by the specific users. In this
# setup, the directory should be writable by both users and should have the
# sticky bit set on it to prevent abuse. Obviously this could be extended to
# as many users as required.
;[myshare]
; comment = Mary's and Fred's stuff
; path = /usr/somewhere/shared
; valid users = mary fred
; public = no
; writable = yes
; printable = no
; create mask = 0765 MASK＝0765
```

启用samba服务：

```
[root@vm51: /etc/samba]#service smb restart
Shutting down SMB services: [ OK ]
Shutting down NMB services: [ OK ]
Starting SMB services: [ OK ]
Starting NMB services: [ OK ]
```

客户端：

```
[root@vm51: /etc/samba]#smbclient -L //10.0.4.51
Password: 浏览
Anonymous login successful
Domain=[MYGROUP] OS=[Unix] Server=[Samba 3.0.23c-2]
Sharename Type Comment
--------- ---- -------
IPC$ IPC IPC Service (Samba Server)
Anonymous login successful
Domain=[MYGROUP] OS=[Unix] Server=[Samba 3.0.23c-2]
Server Comment
--------- -------
VM51 Samba Server
Workgroup Master
--------- -------
MYGROUP
[root@vm51: /etc/samba]#smbclient //10.0.4.51/option
Password:
Anonymous login successful
Domain=[MYGROUP] OS=[Unix] Server=[Samba 3.0.23c-2]
tree connect failed: NT_STATUS_BAD_NETWORK_NAME
[root@vm51: /etc/samba]#smbpasswd -a tq 加用户身份
New SMB password:
Retype new SMB password:
Added user tq.
[root@vm51: /etc/samba]#smbclient //10.0.4.51/option -U tq
Password:
Domain=[VM51] OS=[Unix] Server=[Samba 3.0.23c-2]
tree connect failed: NT_STATUS_BAD_NETWORK_NAME
```

### 二、FTP服务器：

FTP：文件传输协议。

21端口：传控制指令。

20端口：传数据。

特点： ⑴ 支持用户验证；

⑵ 支持匿名用户；

⑶ 对用户做访问控制。

安装包：（1个包）

```
[root@vm51: /etc/vsftpd]#rpm -qf vsftpd.conf
vsftpd-2.0.5-10.el5
```

主控配置文件：`/etc/vsftpd/vsftpd.conf`

```
[root@vm51: /etc/vsftpd]#vim vsftpd.conf
# Example config file /etc/vsftpd/vsftpd.conf
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
anonymous_enable=YES
#
# Uncomment this to allow local users to log in.
local_enable=YES
#
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
#anon_mkdir_write_enable=YES 要开启匿名可写，要开启此两项。
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/vsftpd.log
#
# If you want, you can have your log file in standard ftpd xferlog format
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
listen=YES
#
# This directive enables listening on IPv6 sockets. To listen on IPv4 and IPv6
# sockets, you must run two copies of vsftpd whith two configuration files.
# Make sure, that one of the listen options is commented !!
#listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
```

1． 基于IP、域名的访问控制：tcp_wrappers 。

2． 基于（系统）用户的访问控制。

```
[root@vm51: /etc/vsftpd]#service vsftpd restart
Shutting down vsftpd: [ OK ]
Starting vsftpd for vsftpd: [ OK ]
[root@vm51: /etc/vsftpd]#lftp 10.0.4.51
lftp 10.0.4.51:~> ls
drwxr-xr-x 2 0 0 4096 Jan 17 2007 pub
[root@vm51: /etc/vsftpd]#lftp tq@10.0.4.51 tq为系统以有的用户
Password:
lftp tq@10.0.4.51:~> ls
lftp tq@10.0.4.51:~>
[root@vm51: /etc/vsftpd]#vim user_list
```

此文件内的用户，为不允许用户列表。由FTP支持。

```
# vsftpd userlist
# If userlist_deny=NO, only allow users in this file
# If userlist_deny=YES (default), never allow users in this file, and
# do not even prompt for a password.
# Note that the default vsftpd pam config also checks /etc/vsftpd/ftpusers
# for users that are denied.
root
bin
daemon
adm
lp
sync
shutdown
halt
mail
news
uucp
operator
games
nobody
[root@vm51: /etc/vsftpd]#vim ftpusers
```

此文件内的用户名，为不允许用户列表。由pam支持。

```
[root@vm51: ~]#vim /etc/pam.d/vsftpd
#%PAM-1.0
session optional pam_keyinit.so force revoke
auth required pam_listfile.so item=user sense=deny file=/etc/vsftpd/ftpusers onerr=succeed
auth required pam_shells.so
auth include system-auth
account include system-auth
session include system-auth
session required pam_loginuid.so
# Users that are not allowed to login via ftp
root
bin
daemon
adm
lp
sync
shutdown
halt
mail
news
uucp
operator
games
nobody
```

3. 资源限制（限速）

具体看 ＃`man 5 vsftp.conf`

参数有：

anon_max_rate

connect_timeout

data_connection_timeout

local_max_rate

max_clients

max_per ip

等等……

说明：共享在/etc/passwd文件中ftp用户的主目录中，

ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin

### 三、NFS服务器：

NFS：2049端口

RPC：111端口

![clip_image002](https://o8foyu42q.qnssl.com/tq_notes/linux-file-servers/clip_image002.gif)

安装包：（2个包）

```
[root@vm51: /etc]#rpm -qf `which exportfs`
nfs-utils-1.0.9-16.el5
[root@vm51: /etc]#rpm -qf `which portmap`
portmap-4.0-65.2.2.1
```

步骤：

服务器端：

⑴ [root@vm51: ~]#`service portmap start`

Starting portmap: [ OK ]

⑵ 配置NFS主控配置文件`/etc/exports`。

```
[root@vm51: ~]#vim /etc/exports
/tmp *(ro,sync) 10.0.4.0/24
```

说明：常用属性有：

no_root_squash 保留创建者UID、GID。

all_squash,anonuid=500,anongid=500 创建文件的UID、GID为500。

等等……

⑶ 重启nfs

```
[root@vm51: ~]#service nfs restart
Shutting down NFS mountd: [ OK ]
Shutting down NFS daemon: [ OK ]
Shutting down NFS quotas: [ OK ]
Shutting down NFS services: [ OK ]
Starting NFS services: [ OK ]
Starting NFS quotas: [ OK ]
Starting NFS daemon: [ OK ]
Starting NFS mountd: [ OK ]
```

⑷ 查看资源。

```
[root@vm51: ~]#showmount -e
Export list for vm51.uplooking.com:
/tmp (everyone)
[root@vm51: ~]#showmount -a 查看全部资源
All mount points on vm51.uplooking.com:
*,10.0.4.0/24:/tmp
10.0.4.5:*,10.0.4.0/24
```

客户端：

挂载共享目录即可。

```
[root@vm5: ~]#mount 10.0.4.51:/tmp/ /mnt
[root@vm5: ~]#cd /mnt
[root@vm5: /mnt]#ls
gconfd-root mapping-tq sealert.log
gnome-system-monitor.root.265470486 scim-panel-socket:0-root sound-juicer.root.166966676
mapping-root scim-panel-socket:0-tq ssh-nKDsv25798
```

-- The End --
