---
title: GP的filespace与tablesapce
date: 2017-03-15 23:00:00
categories:
- Greenplum
tags:
- Greenplum
---



## 1. filespace与tablespace

### 1.1 filespace
filespace是GP中特有，是GP的一个目录位置，可以多个tablespace使用同一个filespace. 

greenplum为什么会引入filespace的概念？

因为主机目录结构可能不一样，所以原有的目录结构式的方法来创建表空间，可能不够灵活。查询filespace与目录的对应关系：
```
select a.dbid,a.content,a.role,a.port,a.hostname,b.fsname,c.fselocation from gp_segment_configuration a,pg_filespace b,pg_filespace_entry c where a.dbid=c.fsedbid and b.oid=c.fsefsoid order by content;
```
filespace查询、更改命令：
```
gpfilespace --showtempfilespace {<filespace_name>|default}    #查询
gpfilespace --movetempfilespace {<filespace_name>|default}    #更改临时文件filespace
gpfilespace --movetransfilespace {<filespace_name>|default}   #更改事务文件filespace
```

#### 1.1.1 tempfilespace
tempfilespace存放临时文件、临时表，是全局管理的，也就是说整个GP集群的临时文件是放在一个地方(tempfilespace)。

默认情况下临时文件（例如排序，哈希，产生的work file）和临时表是放在默认的表空间pg_default下面的。

如果需要更改，通过命令：gpfilespace --movetempfilespace {<filespace_name>|default}

tempfilespace用来存储临时表或临时表的索引，以及执行SQL时可能产生的临时文件例如排序，聚合，哈希等。

为了提高性能，一般建议将临时表空间放在SSD或者IOPS，以及吞吐量较高的分区中。

#### 1.1.2 transfilespace
transfilespace存放业务数据，临时表、临时文件以为的。可以有多个。

### 1.2 tablespace
tablespace是数据存放的位置，创建tablespace时要指定filespace，系统初始化后有pg_default和pg_global两个表空间。

创建数据库和创建表时都可以指定使用的表空间。

在创建数据库时指定的是该数据库默认使用的表空间，若不指定则默认表空间则使用pg_default。

在创建表时还可以指定表空间，若不指定则使用创建数据库时指定的默认表空间。同一数据库下的表可以是存储在不同表空间。

## 2. 创建filespace
1、生成配置文件
```
[gpadmin@m1 ~]$ gpfilespace -o filespace_test
20170517:16:06:04:007408 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-
A tablespace requires a file system location to store its database
files. A filespace is a collection of file system locations for all components
in a Greenplum system (primary segment, mirror segment and master instances).
Once a filespace is created, it can be used by one or more tablespaces.


20170517:16:06:04:007408 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-getting config
Enter a name for this filespace
> zly_filespace

Checking your configuration:
Your system has 2 hosts with 4 primary and 4 mirror segments per host.
Your system has 1 hosts with 0 primary and 0 mirror segments per host.

Configuring hosts: [c2.adb.g1.com, c1.adb.g1.com]

Please specify 4 locations for the primary segments, one per line:
primary location 1> /gpadmin/gp_pri_filespc
[Error] c2.adb.g1.com: /gpadmin/gp_pri_filespc : No such file or directory

primary location 1> /gp_pri_filespc
[Error] c2.adb.g1.com: /gp_pri_filespc : No write permissions

primary location 1> /gp_pri_filespc
primary location 2> /gp_pri_filespc
primary location 3> /gp_pri_filespc
primary location 4> /gp_pri_filespc

Please specify 4 locations for the mirror segments, one per line:
mirror location 1> /gp_mir_filespc
mirror location 2> /gp_mir_filespc
mirror location 3> /gp_mir_filespc
mirror location 4> /gp_mir_filespc

Configuring hosts: [m1.adb.g1.com]

Enter a file system location for the master
master location> /gp_master_filespc
20170517:16:14:39:007408 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-Creating configuration file...
20170517:16:14:39:007408 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-[created]
20170517:16:14:39:007408 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-
To add this filespace to the database please run the command:
   gpfilespace --config /home/gpadmin/filespace_test

```
运行命令，交互式输入filespace在各个机器上的位置，生成创建filespace的配置文件。

如果计算节点上，每台机器有4个pimary segment, 4个mirror segment, 则会让你输入4个primary segment的目录，4个mirror segment的目录。

无论集群当前是否已配置standby，只会让你输入一个master的目录。在输入回车后，会检查对应机器上是否有该目录，gpadmin用户是否有写权限。

因此，运行该命令前，需要先在master,standby(如果有),segment机器上创建目录，并用户及用户组权限改为gpadmin.

2、查看配置文件
```
[gpadmin@m1 ~]$ cat zlytest_filespace 
filespace:zly_filespace
m1.adb.g1.com:1:/gp_master_filespc/gpseg-1
m2.adb.g1.com:18:/gp_master_filespc/gpseg-1
c2.adb.g1.com:6:/gp_pri_filespc/gpseg4
c2.adb.g1.com:7:/gp_pri_filespc/gpseg5
c2.adb.g1.com:8:/gp_pri_filespc/gpseg6
c2.adb.g1.com:9:/gp_pri_filespc/gpseg7
c2.adb.g1.com:10:/gp_mir_filespc/gpseg0
c2.adb.g1.com:11:/gp_mir_filespc/gpseg1
c2.adb.g1.com:12:/gp_mir_filespc/gpseg2
c2.adb.g1.com:13:/gp_mir_filespc/gpseg3
c1.adb.g1.com:2:/gp_pri_filespc/gpseg0
c1.adb.g1.com:3:/gp_pri_filespc/gpseg1
c1.adb.g1.com:4:/gp_pri_filespc/gpseg2
c1.adb.g1.com:5:/gp_pri_filespc/gpseg3
c1.adb.g1.com:14:/gp_mir_filespc/gpseg4
c1.adb.g1.com:15:/gp_mir_filespc/gpseg5
c1.adb.g1.com:16:/gp_mir_filespc/gpseg6
c1.adb.g1.com:17:/gp_mir_filespc/gpseg7
```
这是有standby生成的配置文件，每台计算节点上4个primary segment，4个mirror segment

配置文件格式为：
```
filespace:<filespace_name>
<hostname>:<dbid>:/<filesystem_dir>/<seg_datadir_name>
   ...
```

3、创建filespace
```
# 没有在standby上创建目录，于是运行创建filespace报错
[gpadmin@m1 ~]$ gpfilespace --config /home/gpadmin/zlytest_filespace
20170517:17:00:12:023030 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-
A tablespace requires a file system location to store its database
files. A filespace is a collection of file system locations for all components
in a Greenplum system (primary segment, mirror segment and master instances).
Once a filespace is created, it can be used by one or more tablespaces.


20170517:17:00:12:023030 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-getting config
Reading Configuration file: '/home/gpadmin/zlytest_filespace'
20170517:17:00:12:023030 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-Performing validation on paths
....................20170517:17:00:12:023030 gpfilespace:m1.adb.g1.com:gpadmin-[ERROR]:-m2.adb.g1.com: /gp_master_filespc : No such file or directory

# standby上创建目录后运行成功
[root@m2 ~]# mkdir /gp_master_filespc
[root@m2 ~]# chown -R gpadmin:gpadmin /gp_master_filespc/

[gpadmin@m1 ~]$ gpfilespace --config /home/gpadmin/zlytest_filespace
20170517:17:02:45:023257 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-
A tablespace requires a file system location to store its database
files. A filespace is a collection of file system locations for all components
in a Greenplum system (primary segment, mirror segment and master instances).
Once a filespace is created, it can be used by one or more tablespaces.


20170517:17:02:45:023257 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-getting config
Reading Configuration file: '/home/gpadmin/zlytest_filespace'
20170517:17:02:45:023257 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-Performing validation on paths
..............................................................................

20170517:17:02:45:023257 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-Connecting to database
20170517:17:02:45:023257 gpfilespace:m1.adb.g1.com:gpadmin-[INFO]:-Filespace "zly_filespace" successfully created
```

4、查看filespace
```
postgres=# select * from pg_filespace;
    fsname     | fsowner 
---------------+---------
 pg_system     |      10
 zly_filespace |      10
(2 rows)

# master目录
[root@m1 ~]# ls /gp_master_filespc/
gpseg-1

# primary segment目录
[root@c1 ~]# ls /gp_pri_filespc/
gpseg0  gpseg1  gpseg2  gpseg3

# mirror segment目录
[root@c1 ~]# ls /gp_mir_filespc/
gpseg4  gpseg5  gpseg6  gpseg7
```
## 3. tablespace的使用
1、创建tablespace
```
postgres=# create TABLESPACE zly_tablespace FILESPACE zly_filespace;
CREATE TABLESPACE
```

2、查看tablespace
```
postgres=# select * from pg_tablespace ;
    spcname     | spcowner | spclocation | spcacl | spcprilocations | spcmirlocations | spcfsoid 
----------------+----------+-------------+--------+-----------------+-----------------+----------
 pg_default     |       10 |             |        |                 |                 |     3052
 pg_global      |       10 |             |        |                 |                 |     3052
 zly_tablespace |       10 |             |        |                 |                 |    16423
(3 rows)
```

、创建数据库并指定默认的tablespace
```
postgres=# create database zlytest TABLESPACE zly_tablespace;
CREATE DATABASE
```

4、创建表
```
zlytest=# create table test1(id int, name varchar(25));
zlytest=# create table test2(id int, name varchar(25)) tablespace pg_default;
zlytest=# create table test3(id int, name varchar(25)) tablespace zly_tablespace;
```

5、查看表所在tablespace
```
zlytest=# select * from pg_tables where schemaname='public';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers 
------------+-----------+------------+------------+------------+----------+-------------
 public     | test1     | gpadmin    |            | f          | f        | f
 public     | test2     | gpadmin    | pg_default | f          | f        | f
 public     | test3     | gpadmin    |            | f          | f        | f
(3 rows)
```
test1和test3使用该数据库默认的表空间zly_tablespace