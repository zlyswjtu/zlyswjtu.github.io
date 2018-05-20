---
title: Postgresql客户端认证
date: 2017-09-24 23:00:00
categories:
- Postgresql
tags:
- Postgresql
---

pg_hba.conf配置数据连接认证方式，内容如下
```
#TYPE  DATABASE  USER  CIDR-ADDRESS  METHOD
 
#"local" is for Unix domain socket connections only
local    all      all                 ident
 
#IPv4 local connections:
host     all      all   127.0.0.1/32  md5
 
#IPv6 local connections:
host     all      all   ::1/128       md5
```
TYPE定义了多种连接PostgreSQL的方式，分别是：“local”使用本地unix套接字，“host”使用TCP/IP连接（包括SSL和非SSL），“host”结合“IPv4地址”使用IPv4方式，结合“IPv6地址”则使用IPv6方式，“hostssl”只能使用SSL TCP/IP连接，“hostnossl”不能使用SSL TCP/IP连接。

DATABASE指定哪个数据库，多个数据库，库名间以逗号分隔。“all”只有在没有其他的符合条目时才代表“所有”，如果有其他的符合条目则代表“除了该条之外的”，因为“all”的优先级最低。如下例：
```
local    db1    user1    reject
local    all      all        ident
```
这两条都是指定local访问方式，因为前一条指定了特定的数据库db1，所以后一条的all代表的是除了db1之外的数据库，同理用户的all也是这个道理。

USER指定哪个数据库用户（PostgreSQL正规的叫法是角色，role）。多个用户以逗号分隔。

CIDR-ADDRESS项local方式不必填写，该项可以是IPv4地址或IPv6地址，可以定义某台主机或某个网段。

METHOD指定如何处理客户端的认证。常用的有ident，password，md5，trust，reject。

1. ident认证
ident是默认的local认证方式，凡是能正确登录服务器的操作系统用户（注：不是数据库用户）就能使用本用户映射的数据库用户不需密码登录数据库。用户映射文件为pg_ident.conf，这个文件记录着与操作系统用户匹配的数据库用户，如果某操作系统用户在本文件中没有映射用户，则默认的映射数据库用户与操作系统用户同名。比如，服务器上有名为user1的操作系统用户，同时数据库上也有同名的数据库用户，user1登录操作系统后可以直接输入psql database，以user1数据库用户身份登录数据库且不需密码。
```
#pg_hba.conf
local    all         all              ident   map=mapzly

#pg_ident.conf
mapzly          root                    gpadmin
```
这个配置的含义:
当客户端使用unix socket连接数据库时，使用ident认证。 当客户端的OS用户是root时，允许它以数据库用户gpadmin连接数据库。
```
[root@m1 ~]# whoami
root

[root@m1 ~]# psql -U gpadmin -d postgres
psql (8.3.23)
Type "help" for help.

postgres=# \q
```
如果不存在这个映射，将报如下错误：
```
[root@m1 ~]# whoami
root

[root@m1 ~]# psql -U gpadmin -d postgres
psql: FATAL:  Ident authentication failed for user "gpadmin"
```

2. password
使用密码进行验证，密码是明文传送给数据库的，建议不要在生产环境中使用。

3. md5
是常用的密码认证方式，密码是以md5形式传送给数据库。

4. trust
不进行权限验证，允许连接。

5. reject
拒绝连接。

