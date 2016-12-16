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

### 一、apache服务器：

安装包：（一个包）

[root@vm51: ~]#rpm -qa | grep httpd

httpd-manual-2.2.3-6.el5 安装手册包后，手册在/var/www/manual/index.html

system-config-httpd-1.3.3.1-1.el5

httpd-2.2.3-6.el5

主控配置文件：/etc/httpd/conf/httpd.conf

/etc/httpd/目录，为配置文件的目录

/etc/httpd/modules/ 模块

/etc/httpd/run/ PID

/etc/httpd/conf/httpd.conf 主控配置文件

/etc/httpd/conf.d/ 配置文件

/etc/httpd/logs/ 日志

[root@vm51: ~]#vim /etc/httpd/conf/httpd.conf

ServerTokens OS 平台版本。OS：比较详细的级别。

ServerRoot "/etc/httpd" 配置文件的路径

PidFile run/httpd.pid 记录下其PID ，可用此PID关此httpd进程；还可以为文件加锁，当httpd运行时，别的进程不那改它。

Timeout 120 超时时间

＃长连接：TCP三次握手后，可发多个http请求叫长连接；只发一个http请求，叫短连接。

KeepAlive Off 默认：关闭长连接。但，一般，打开长连接比较好（提高效率）。

MaxKeepAliveRequests 100 最大长连接请求数（正常不会超过100）如：论坛，可再小此；用于下载，要大些。

KeepAliveTimeout 15 长连接的超时时间

# prefork MPM 进程工作方式（默认启用进程方式）

# StartServers: number of server processes to start

# MinSpareServers: minimum number of server processes which are kept spare

# MaxSpareServers: maximum number of server processes which are kept spare

# ServerLimit: maximum value for MaxClients for the lifetime of the server

# MaxClients: maximum number of server processes allowed to start

# MaxRequestsPerChild: maximum number of requests a server process serves

<IfModule prefork.c> 判断是否有perfork.c模块

StartServers 8 第一次同时并发打开8 个子进程

MinSpareServers 5 当空闲连接数小于5时，再打开1个子进程。保证至少有5个空闲连接数。

MaxSpareServers 20 最大空闲连接数。

ServerLimit 256 服务端最多开多少个进程数（包括：空闲和非空闲进程）

MaxClients 256 客户端最多个数。此数值同ServerLimit的数值。

MaxRequestsPerChild 4000 每个进程最大http请求数。到4000个http请求就杀掉此进程。（太大会占用很大内存）

</IfModule>

# worker MPM 线程工作方式。在linux中，线程是使用相同的系统环境的进程。线程使用系统资源的消耗要小一些。

# StartServers: initial number of server processes to start

# MaxClients: maximum number of simultaneous client connections

# MinSpareThreads: minimum number of worker threads which are kept spare

# MaxSpareThreads: maximum number of worker threads which are kept spare

# ThreadsPerChild: constant number of worker threads in each server process

# MaxRequestsPerChild: maximum number of requests a server process serves

<IfModule worker.c>

StartServers 2 第一次开2个进程。

MaxClients 150 最多开150个线程。（此时，每个进程最多75个线程）

MinSpareThreads 25 最小空闲线程数。

MaxSpareThreads 75 最大空闲线程数。

ThreadsPerChild 25 每开一个进程开25个线程。当小于25时，再开一个线程。

MaxRequestsPerChild 0 最多开几个进程数。为0：不限制进程数。

</IfModule>

Listen 80 监听80端口。（可Listen IP地址：80

如：Listen 127.0.0.1:80）

LoadModule auth_basic_module modules/mod_auth_basic.so

＃LoadModule 模块名 加载外部模块

Include conf.d/*.conf 包含此目录下的所有.conf的配置文件

User apache 用户身份

Group apache 组身份

### Section 2: 'Main' server configuration

ServerAdmin root@localhost

UseCanonicalName Off

DocumentRoot "/var/www/html"

<Directory />

Options FollowSymLinks

AllowOverride None

</Directory>

<Directory "/var/www/html">

Options Indexes FollowSymLinks

AllowOverride None

Order allow,deny

Allow from all

</Directory>

# AccessFileName: The name of the file to look for in each directory 文件限制

# for additional configuration directives. See also the AllowOverride

# directive.

#

AccessFileName .htaccess

#

# The following lines prevent .htaccess and .htpasswd files from being

# viewed by Web clients.

#

<Files ~ "^\.ht">

Order allow,deny

Deny from all

</Files>

CustomLog logs/access_log combined 日志

Alias /icons/ "/var/www/icons/" 别名

<Directory "/var/www/icons">

Options Indexes MultiViews

AllowOverride None

Order allow,deny

Allow from all

</Directory>

ScriptAlias /cgi-bin/ "/var/www/cgi-bin/" CGI动态网页结构

<Directory "/var/www/cgi-bin">

AllowOverride None

Options None

Order allow,deny

Allow from all

</Directory>

AddLanguage ca .ca 加语言

AddLanguage cs .cz .cs

AddLanguage da .dk

AddLanguage de .de

AddLanguage el .el

AddLanguage en .en

AddLanguage eo .eo

AddLanguage es .es

AddLanguage et .et

AddLanguage fr .fr

AddLanguage he .he

AddLanguage hr .hr

AddLanguage it .it

AddLanguage ja .ja

AddLanguage ko .ko

AddLanguage ltz .ltz

AddLanguage nl .nl

AddLanguage nn .nn

AddLanguage no .no

AddLanguage pl .po

AddLanguage pt .pt

AddLanguage pt-BR .pt-br

AddLanguage ru .ru

AddLanguage sv .sv

AddLanguage zh-CN .zh-cn

AddLanguage zh-TW .zh-tw

LanguagePriority en ca cs da de el eo es et fr he hr it ja ko ltz nl nn no pl pt pt-BR ru sv zh-CN zh-TW 语言优先级

AddDefaultCharset UTF-8 默认字符集

### Section 3: Virtual Hosts 虚拟主机

#<VirtualHost *:80>

# ServerAdmin webmaster@dummy-host.example.com

# DocumentRoot /www/docs/dummy-host.example.com

# ServerName dummy-host.example.com

# ErrorLog logs/dummy-host.example.com-error_log

# CustomLog logs/dummy-host.example.com-access_log common

#</VirtualHost>

启用进程或线程的切换：（默认以进程方程）

[root@vm51: ~]#service httpd stop

Stopping httpd: [ OK ]

[root@vm51: /etc/httpd/run]#vim /etc/sysconfig/httpd

# Configuration file for the httpd service.

#

# The default processing model (MPM) is the process-based

# 'prefork' model. A thread-based model, 'worker', is also

# available, but does not work with some modules (such as PHP).

# The service must be stopped before changing this variable.

#

#HTTPD=/usr/sbin/httpd.worker （去掉＃）开启线程方式

#默认用进程方式，HTTPD=/usr/sbin/httpd

# To pass additional options (for instance, -D definitions) to the

# httpd binary at startup, set OPTIONS here.

#

#OPTIONS=

#

# By default, the httpd process is started in the C locale; to

# change the locale in which the server runs, the HTTPD_LANG

# variable can be set.

#

#HTTPD_LANG=C

[root@vm51: ~]#service httpd start

Starting httpd: [ OK ]

查看是进程方式还是线程方式：

[root@vm51: ~]#ps aux | grep httpd

root 27281 0.0 1.0 9256 2772 pts/1 S+ 15:51 0:00 vim /etc/httpd/conf/httpd.conf

root 27546 0.2 3.5 22804 9188 ? Ss 17:05 0:00 /usr/sbin/httpd

apache 27548 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27549 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27550 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27551 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27552 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27553 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27554 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

apache 27555 0.0 1.8 22936 4700 ? S 17:05 0:00 /usr/sbin/httpd

root 27561 0.0 0.2 3880 668 pts/2 R+ 17:06 0:00 grep httpd

注：此时是进程工作方式。有一个root父进程，有8个apache子进程。

/usr/sbin/httpd

[root@vm51: ~]#ps aux | grep httpd

root 27813 1.5 3.1 16728 7988 ? Ss 17:13 0:00 /usr/sbin/httpd.worker

apache 27815 0.3 2.1 293364 5404 ? Sl 17:13 0:00 /usr/sbin/httpd.worker

apache 27816 0.3 2.1 293364 5404 ? Sl 17:13 0:00 /usr/sbin/httpd.worker

root 27873 0.0 0.2 3884 712 pts/2 S+ 17:13 0:00 grep httpd

注：此时是线程工作方式。有一个root父进程，有2个apache子进程。

/usr/sbin/httpd.worker

测试：http://10.0.4.51

clip_image002

### 二、squid代理服务器

安装包：（1个）

[root@vm51: ~]#rpm -qa | grep squid

squid-2.6.STABLE6-3.el5

默认：3128端口

常用加速Web服务，只能加速静态网页。

主控配置文件：/etc/squid/squid.conf

[root@vm51: ~]#vim /etc/squid/squid.conf

# TAG: http_port

http_port 3128 对外监听端口

＃＃＃做代理，缓存加速httpd。

＃＃在httpd.conf中，写Listen 127.0.0.1:80

＃＃此处写，http_port 80 vhost 做虚拟主机加速

# TAG: cache_peer 对等缓存（对内缓存）

# To specify other caches in a hierarchy, use the format:

#

# cache_peer hostname type http_port icp_port [options]

#

# For example,

#

# # proxy icp

# # hostname type port port options

# # -------------------- -------- ----- ----- -----------

# cache_peer parent.foo.net parent 3128 3130 [proxy-only]

# cache_peer sib1.foo.net sibling 3128 3130 [proxy-only]

# cache_peer sib2.foo.net sibling 3128 3130 [proxy-only]

注：类型：parent父子关系；sibling子妹关系。

icp端口：一般为0（不启用）。同步squid之间数据。

options属性：originserver 用作加速器。

proxy-only（默认）只代理。可能更新不及时。

例：cache_peer 127.0.0.1 parent 8080 0 originserver

重启service squid restart 或 killall -1 squid

＃＃＃只做代理。

# TAG: cache_mem (bytes)

#Default:

cache_mem 8 MB

# TAG: cache_swap_low (percent, 0-100)

# TAG: cache_swap_high (percent, 0-100)

#Default:

cache_swap_low 90

cache_swap_high 95

# TAG: cache_dir

cache_dir ufs /var/spool/squid 100 16 256 100：100兆 16：此目录下有16个一级目录 256：此目录下有256个二级目录

# ACCESS CONTROLS 访问列表控制

# -----------------------------------------------------------------------------

# TAG: acl

# acl aclname src ip-address/netmask ... (clients IP address)

# acl aclname src addr1-addr2/netmask ... (range of addresses)

# acl aclname dst ip-address/netmask ... (URL host's IP address)

# acl aclname myip ip-address/netmask ... (local socket IP address)

# acl aclname srcdomain .foo.com ... # reverse lookup, client IP

# acl aclname dstdomain .foo.com ... # Destination server from URL

# acl aclname srcdom_regex [-i] xxx ... # regex matching client name

# acl aclname dstdom_regex [-i] xxx ... # regex matching server

#Recommended minimum configuration:

例：acl tec src 192.168.0.0/255.255.255.0

acl sell src 192.168.1.0/255.255.255.0

acl up dstdomain up.com

acl all src 0.0.0.0/0.0.0.0

acl manager proto cache_object

acl localhost src 127.0.0.1/255.255.255.255

acl to_localhost dst 127.0.0.0/8

acl SSL_ports port 443

acl Safe_ports port 80 # http

acl Safe_ports port 21 # ftp

acl Safe_ports port 443 # https

acl Safe_ports port 70 # gopher

acl Safe_ports port 210 # wais

acl Safe_ports port 1025-65535 # unregistered ports

acl Safe_ports port 280 # http-mgmt

acl Safe_ports port 488 # gss-http

acl Safe_ports port 591 # filemaker

acl Safe_ports port 777 # multiling http

acl CONNECT method CONNECT

#Default:

# http_access deny all

#

#Recommended minimum configuration:

#

# Only allow cachemgr access from localhost

例：http_access allow tec

http_access deny !up

http_access allow sell

http_access allow manager localhost

http_access deny manager

# Deny requests to unknown ports

http_access deny !Safe_ports

# Deny CONNECT to other than SSL ports

http_access deny CONNECT !SSL_ports

# TAG: refresh_pattern 刷新频率（可用正规表达式）

#Suggested default:

refresh_pattern ^ftp: 1440 20% 10080

refresh_pattern ^gopher: 1440 0% 1440

refresh_pattern . 0 20% 4320



透明代理：

设置透明代理，用户根本看不出服务使用了代理服务器。

squid结合iptables设置透明代理。

步骤：（2步）

⑴ 设置squid.conf

[root@vm51: ~]#vim /etc/squid/squid.conf

# options are:

# transparent Support for transparent proxies

# vhost Accelerator using Host directive

# vport Accelerator with IP virtual host support

http_port 3128 transparent

⑵ 设置iptables

[root@vm51: ~]#iptables -t nat -A PREROUTING -s 10.0.4.0/24 -p tcp --dport 80 -j REDIRECT --to-ports 3128

[root@vm51: ~]#iptables -t nat -L

Chain PREROUTING (policy ACCEPT)

target prot opt source destination

REDIRECT tcp -- 10.0.4.0/24 anywhere tcp dpt:http redir ports 3128

Chain POSTROUTING (policy ACCEPT)

target prot opt source destination

Chain OUTPUT (policy ACCEPT)

target prot opt source destination

- The End -
