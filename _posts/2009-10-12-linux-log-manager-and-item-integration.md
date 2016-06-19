---
layout: post
title:  "Linux日志管理及项目集成"
date:   2009-10-12 +0800
categories: Linux
tags: Linux
author: dbtan
---

* content
{:toc}


### 一、日志服务：syslog





```
分为：kernel logger 内核日志；
system logger 系统日志。
[root@vm5: ~]#service syslog restart
Shutting down kernel logger: [ OK ]
Shutting down system logger: [ OK ]
Starting system logger: [ OK ]
Starting kernel logger: [ OK ]
查看内核日志用dmesg命令。
[root@vm5: ~]#dmesg
Linux version 2.6.18-8.el5 (brewbuilder@ls20-bc2-14.build.redhat.com) (gcc version 4.1.1 20070105 (Red Hat 4.1.1-52)) #1 SMP Fri Jan 26 14:15:21 EST 2007
BIOS-provided physical RAM map:
BIOS-e820: 0000000000000000 - 000000000009f800 (usable)
BIOS-e820: 000000000009f800 - 00000000000a0000 (reserved)
BIOS-e820: 00000000000ca000 - 00000000000cc000 (reserved)
BIOS-e820: 00000000000dc000 - 0000000000100000 (reserved)
BIOS-e820: 0000000000100000 - 000000000fef0000 (usable)
BIOS-e820: 000000000fef0000 - 000000000feff000 (ACPI data)
BIOS-e820: 000000000feff000 - 000000000ff00000 (ACPI NVS)
BIOS-e820: 000000000ff00000 - 0000000010000000 (usable)
BIOS-e820: 00000000fec00000 - 00000000fec10000 (reserved)
BIOS-e820: 00000000fee00000 - 00000000fee01000 (reserved)
BIOS-e820: 00000000fffe0000 - 0000000100000000 (reserved)
0MB HIGHMEM available.
256MB LOWMEM available.
found SMP MP-table at 000f6cd0
Using x86 segment limits to approximate NX protection
On node 0 totalpages: 65536
DMA zone: 4096 pages, LIFO batch:0
Normal zone: 61440 pages, LIFO batch:15
DMI present.
Using APIC driver default
ACPI: RSDP (v000 PTLTD ) @ 0x000f6c60
ACPI: RSDT (v001 PTLTD RSDT 0x06040000 LTP 0x00000000) @ 0x0fefab5a
---------------------------------------后面内容省略了，太多了---------------------------------------
/var/log/：登录文件放置的目录。
/var/log/messages：是总管所有登录文件的文件（即：日志文件）。
syslog日志服务的配置文件：/etc/syslog.conf 。
[root@vm5: ~]#vim /etc/syslog.conf
# Log all kernel messages to the console.
# Logging much else clutters up the screen.
#kern.* /dev/console
# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;news.none;authpriv.none;cron.none /var/log/messages
# The authpriv file has restricted access.
authpriv.* /var/log/secure
# Log all the mail messages in one place.
mail.* -/var/log/maillog
注：-表示异步磁盘数据，有用缓存。
# Log cron stuff
cron.* /var/log/cron
# Everybody gets emergency messages
*.emerg *
# Save news errors of level crit and higher in a special file.
uucp,news.crit /var/log/spooler
# Save boot messages also to boot.log
local7.* /var/log/boot.log
#
# INN
#
news.=crit /var/log/news/news.crit
news.=err /var/log/news/news.err
news.notice /var/log/news/news.notice
说明：
日志有：对象.等级
```

![对象.等级](https://o8foyu42q.qnssl.com/tq_notes/linux-log-manager-and-item-integration/%E5%AF%B9%E8%B1%A1-%E7%AD%89%E7%BA%A7.png)

```

news.=crit 就这一级的信息；不加“＝”就从本级到最高级。
news.!crit “！”：取反，除了crit级的信息。
news.*；news.crit；news.err “；”：排除等一个“分号；”后的信息。
*.info;mail.none .none表示不记。
-/var/log/maillog 注：-表示异步磁盘数据，有用缓存。
日志可写到设备上：如：/dev/tty12
日志可写给用户：（三种）
⑴ “用户名”，如：root
⑵ ＠IP地址，如：＠192.168.0.66 表示接收来自192.168.0.66发来的日志，要开启远程管理（加-r） （在客户端）
⑶ ＊，表示给所有用户。
开启日志服务的远程管理功能，在/etc/sysconfig/syslog文件中设置。
[root@vm5: ~]#vim /etc/sysconfig/syslog
# Options to syslogd
# -m 0 disables 'MARK' messages.
# -r enables logging from remote machines
# -x disables DNS lookups on messages recieved with -r
# See syslogd(8) for more details
SYSLOGD_OPTIONS="-m 0 -r -x" 注：-m：MAC -r：开启远程日志 -x：不DNS
# Options to klogd
# -2 prints all kernel oops messages twice; once for klogd to decode, and
# once for processing with 'ksymoops'
# -x disables all klogd processing of oops messages entirely
# See klogd(8) for more details
KLOGD_OPTIONS="-x"
#
SYSLOG_UMASK=077
# set this to a umask value to use for all log files as in umask(1).
# By default, all permissions are removed for "group" and "other".
可以用ps aux | grep syslog 来查看是否开启“日志远程管理”功能。
[root@vm5: ~]#ps aux | grep syslog
root 4338 0.0 0.2 1688 576 ? Ss 05:00 0:00 syslogd -m 0 -r -x
root 4354 0.0 0.2 3884 680 pts/4 S+ 05:01 0:00 grep syslog
例：找本局域网内日志最多的机器。
[root@vm5: ~]#awk '{print $4}' /var/log/messages | sort | uniq -c
297 10.0.4.4 sort：排序 –n:按数字排
413 localhost
1375 vm5
[root@vm5: ~]#awk '{print $4}' /var/log/messages | uniq -c | sort -n
52 vm5 uniq：去除重复行 -c：计数
297 10.0.4.4
413 localhost
544 vm5
779 vm5
[root@vm5: ~]#awk '{print $4}' /var/log/messages | sort | uniq -c | sort -n
297 10.0.4.4
413 localhost
1375 vm5
[root@vm5: ~]#awk '{print $4}' /var/log/messages | sort | uniq -c | sort -nr
1375 vm5
413 localhost
297 10.0.4.4
[root@vm5: ~]#awk '{print $4}' /var/log/messages | sort | uniq -c | sort -nr | head -1
1375 vm5
在/etc/logrotate.d/下，是日志记录的信息。
[root@vm5: /etc/logrotate.d]#ls
acpid cups mgetty ppp rpm sa-update squid tux vsftpd.log
conman httpd named psacct samba setroubleshoot syslog up2date yum
[root@vm5: /etc/logrotate.d]#cat httpd
/var/log/httpd/*log {
missingok
notifempty
sharedscripts
postrotate
/bin/kill -HUP `cat /var/run/httpd.pid 2>/dev/null` 2> /dev/null || true
endscript
}
配置文件在/etc/logrotate.conf中，用来设置日志来如何记录。
[root@vm5: ~]#vim /etc/logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly
# keep 4 weeks worth of backlogs
rotate 4
# create new (empty) log files after rotating old ones
create
# uncomment this if you want your log files compressed
#compress
# RPM packages drop log rotation information into this directory
include /etc/logrotate.d
# no packages own wtmp -- we'll rotate them here
/var/log/wtmp {
monthly
create 0664 root utmp
rotate 1
}
# system-specific logs may be also be configured here.
在计划任务中有：/etc/cron.daily/logrotate文件。
[root@vm5: ~]#vim /etc/cron.daily/logrotate
#!/bin/sh
/usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
/usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
logger命令，常在脚本中用，使用脚本进入日志。
[root@vm5: ~]#logger -p local0.info "aaaaaa" 注：-p：加对象.级别
[root@vm5: ~]#tail -1 /var/log/messages
Feb 6 05:49:07 vm5 root: aaaaaa
用户名
[root@vm5: ~]#logger -p local.info -t abc "bbbbbb" 注：-t：加标签。
[root@vm5: ~]#tail -1 /var/log/messages
Feb 6 05:51:25 vm5 abc: bbbbbb
标签
日志相关：用iptables记日志。
-j后加LOG --log-level info
日志级别
例：
[root@vm5: ~]#iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 80 -j LOG --log-level info
[root@vm5: ~]#tail -1 /var/log/messages
Feb 6 05:54:02 vm5 kernel: ip_tables: (C) 2000-2006 Netfilter Core Team
对象名
∵iptables是由内核直接的，只是kernel对象。
∴iptables只能指定其级别。
```

### 二、日志项目集成：

```
项目说明：多台Web服务用用LVS进行负载均衡，并且多台Web服务器的apache日志由独立一台日志服务器来记录到syslog日志中。（这里主要介绍如何使用syslog日志服务远程管理apache日志）
```

![logger](https://o8foyu42q.qnssl.com/tq_notes/linux-log-manager-and-item-integration/logger.gif)

```
∵apache本身有一套日志系统，记录在/var/log/httpd/中。现在想用独立的一台syslog日志服务器记录apache日志信息。
∴用“管道”：| logger
在各台Web服务器设置：
⑴ 上加-r参数，开启远程日志管理。
[root@vm5: ~]#vim /etc/sysconfig/syslog
---------------------------------------前略---------------------------------------
SYSLOGD_OPTIONS="-m 0 -r -x" 注：-m：MAC -r：开启远程日志 -x：不DNS
---------------------------------------后略---------------------------------------
⑵ 在apache的配置文件/etc/httpd/conf/httpd.conf中，增加| logger 。
[root@vm5: ~]#vim /etc/httpd/conf/httpd.conf
---------------------------------------前略---------------------------------------
# For a single logfile with access, agent, and referer information
# (Combined Logfile Format), use the following directive:
#
CustomLog logs/access_log combined
CustomLog "| logger -p info" combined
---------------------------------------后略---------------------------------------
此时，就可以在独立的syslog日志服务器上，查看到各台Web服务器的日志了！！！
```

-- The End --
