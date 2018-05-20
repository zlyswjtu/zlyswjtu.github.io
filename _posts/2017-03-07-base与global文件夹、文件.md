---
title: GP的base与global目录
date: 2017-03-07 23:00:00
categories:
- Greenplum
tags:
- Greenplum
---



src/backend/commands/tablespace.c中的注释：

``` 
 * There are two tablespaces created at initdb time: pg_global (for shared
 * tables) and pg_default (for everything else).  For backwards compatibility
 * and to remain functional on platforms without symlinks, these tablespaces
 * are accessed specially: they are respectively
 *          $PGDATA/global/relfilenode
 *          $PGDATA/base/dboid/relfilenode
```
初始时有两个tablespace，pg_default和pg_global, pg_default是默认表空间，存储模板数据库template0,template1和postgres，同时临时文件也会存在默认表空间$PGDATA/base/dboid/pg_tmp(如果不更改临时表的filespace,临时表filespace可以通命令gpfilespace更改)。
pg_gloab用于存储共享的系统元数据，例如pg_database等。

gloab/pg_database文件存放了数据库的信息
```
[gpadmin@m1 global]$ cat pg_database 
"template1" 1 1663 677
"template0" 12058 1663 677
"postgres" 12059 1663 677
"zlytest" 16384 1663 677
"zlytest_copy" 16470 1663 677
```
base/目录下存放默认表空间pg_default里得各个数据库，每个文件夹代表一个数据库，ID可以从上面的pg_database中查到，数据库zlytest的编号是16384
```
[gpadmin@m1 gpseg-1]$ ls base
1  12058  12059  16384  16470
```
通过查询select relfilenode, relname from pg_class where relname = 'tablename';，可以查找到表、索引的relfilenode，这个relfilenode就是存储数据的文件名。
```
zlytest=# select relfilenode, relname from pg_class where relname = 'supplier';
 relfilenode | relname  
-------------+----------
       32845 | supplier
(1 row)

zlytest=# select relfilenode, relname from pg_class where relname = 'supplier_row';
 relfilenode |   relname    
-------------+--------------
       32877 | supplier_row
(1 row)

zlytest=# select relfilenode, relname from pg_class where relname = 'supplier_col';
 relfilenode |   relname    
-------------+--------------
       32913 | supplier_col
(1 row)
```
在master只存储元数据，是不存储业务数据的，于是对应的文件大小为0。业务数据存储在segment上，于是对应的文件大小不为0，且较大的heap表会分成几个文件存储，apendonly表也是分几个文件存储的。
```
#master上
[gpadmin@m1 gpseg-1]$ ll -h base/16384/32845* base/16384/32877* base/16384/32913*
-rw------- 1 gpadmin gpadmin 0 Feb 15 15:46 base/16384/32845
-rw------- 1 gpadmin gpadmin 0 Feb 15 16:00 base/16384/32877
-rw------- 1 gpadmin gpadmin 0 Feb 15 16:04 base/16384/32913

#segment上
[root@seg1 gpseg0]# ll -h base/16384/32845* base/16384/32877* base/16384/32913*
-rw------- 1 gpadmin gpadmin 661M Feb 17 13:34 base/16384/32845
-rw------- 1 gpadmin gpadmin    0 Feb 15 16:00 base/16384/32877
-rw------- 1 gpadmin gpadmin 301M Feb 15 16:04 base/16384/32877.1
-rw------- 1 gpadmin gpadmin    0 Feb 15 16:04 base/16384/32913
-rw------- 1 gpadmin gpadmin 7.2M Feb 15 16:11 base/16384/32913.1
-rw------- 1 gpadmin gpadmin  47M Feb 15 16:11 base/16384/32913.129
-rw------- 1 gpadmin gpadmin  47M Feb 15 16:11 base/16384/32913.257
-rw------- 1 gpadmin gpadmin 7.2M Feb 15 16:11 base/16384/32913.385
-rw------- 1 gpadmin gpadmin  29M Feb 15 16:11 base/16384/32913.513
-rw------- 1 gpadmin gpadmin  17M Feb 15 16:11 base/16384/32913.641
-rw------- 1 gpadmin gpadmin 114M Feb 15 16:11 base/16384/32913.769
```
supplier表是heap表，文件32845，由于数据量比较小，只对应一个文件，而数据量比较大的heap表，为了防止文件系统对磁盘文件大小的限制而导致的写入失败，每1G就划分1个文件，xx.1, xx.2的方式命名，例如：
```
zlytest=# select relfilenode, relname from pg_class where relname = 'part';
 relfilenode | relname 
-------------+---------
       32949 | part
(1 row)

[root@seg1 gpseg0]# ll -h base/16384/32949*
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 16:47 base/16384/32949
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 16:58 base/16384/32949.1
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 17:09 base/16384/32949.2
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 17:20 base/16384/32949.3
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 17:31 base/16384/32949.4
-rw------- 1 gpadmin gpadmin 1.0G Feb 15 18:13 base/16384/32949.5
-rw------- 1 gpadmin gpadmin 304M Feb 15 18:15 base/16384/32949.6
```
从上面的两个例子可以看出heap表的第一个数据文件大小是不为0的，而appendonly表第一个数据文件大小为0，后续文件才真正存数据。列存表与行存不同，列存表每一列存一个文件，例如supplier_col表有7个列，于是数据存了7个文件，.后面的应该是表示列ID？
```
zlytest=# \d supplier_col
                            Append-Only Columnar Table "public.supplier_col"
   Column    |          Type          |                            Modifiers                             
-------------+------------------------+------------------------------------------------------------------
 s_suppkey   | integer                | not null default nextval('supplier_col_s_suppkey_seq'::regclass)
 s_name      | character(25)          | 
 s_address   | character varying(40)  | 
 s_nationkey | integer                | not null
 s_phone     | character(15)          | 
 s_acctbal   | numeric                | 
 s_comment   | character varying(101) | 
Checksum: t
Distributed by: (s_suppkey)

-rw------- 1 gpadmin gpadmin    0 Feb 15 16:04 base/16384/32913
-rw------- 1 gpadmin gpadmin 7.2M Feb 15 16:11 base/16384/32913.1
-rw------- 1 gpadmin gpadmin  47M Feb 15 16:11 base/16384/32913.129
-rw------- 1 gpadmin gpadmin  47M Feb 15 16:11 base/16384/32913.257
-rw------- 1 gpadmin gpadmin 7.2M Feb 15 16:11 base/16384/32913.385
-rw------- 1 gpadmin gpadmin  29M Feb 15 16:11 base/16384/32913.513
-rw------- 1 gpadmin gpadmin  17M Feb 15 16:11 base/16384/32913.641
-rw------- 1 gpadmin gpadmin 114M Feb 15 16:11 base/16384/32913.769
```
appendonly列存表是否按文件大小分段？这个例子中有一列对应的文件大小超过1G

更新：appendonly最多128个segment file，其中0号segment file不用于存储数据，除非utility模式下执行增删改，1-127并发使用
AO行存表对文件大小没有限制，只对行数有限制，在AO表commit的时候更新isfull状态
如果total_tupount（有效+无效）超过0.9 * 1099511627775，isfull置为true

```
zlytest=# select relfilenode, relname from pg_class where relname = 'part_col';
 relfilenode | relname  
-------------+----------
       33017 | part_col
(1 row)

[root@seg1 gpseg0]# ll -h base/16384/33017*
-rw------- 1 gpadmin gpadmin    0 Feb 15 16:24 base/16384/33017
-rw------- 1 gpadmin gpadmin 144M Feb 17 16:46 base/16384/33017.1
-rw------- 1 gpadmin gpadmin 520M Feb 17 16:46 base/16384/33017.1025
-rw------- 1 gpadmin gpadmin 1.2G Feb 17 16:46 base/16384/33017.129
-rw------- 1 gpadmin gpadmin 932M Feb 17 16:46 base/16384/33017.257
-rw------- 1 gpadmin gpadmin 394M Feb 17 16:46 base/16384/33017.385
-rw------- 1 gpadmin gpadmin 774M Feb 17 16:46 base/16384/33017.513
-rw------- 1 gpadmin gpadmin 144M Feb 17 16:46 base/16384/33017.641
-rw------- 1 gpadmin gpadmin 394M Feb 17 16:46 base/16384/33017.769
-rw------- 1 gpadmin gpadmin 322M Feb 17 16:46 base/16384/33017.897
```
同样，appendonly行存表文件大小也超过1G
```
zlytest=# select relfilenode, relname from pg_class where relname = 'part_row';
 relfilenode | relname  
-------------+----------
       32981 | part_row
(1 row)

ll -h base/16384/32981*
-rw------- 1 gpadmin gpadmin    0 Feb 15 16:24 base/16384/32981
-rw------- 1 gpadmin gpadmin 5.6G Feb 17 17:58 base/16384/32981.1
```
索引文件在master上和segment上大小都不为0.
```
zlytest=# \d+ window_test 
               Table "public.window_test"
 Column  |  Type   | Modifiers | Storage  | Description 
---------+---------+-----------+----------+-------------
 id      | integer |           | plain    | 
 name    | text    |           | extended | 
 subject | text    |           | extended | 
 score   | numeric |           | main     | 
Indexes:
    "idx_id" btree (id)
Has OIDs: no
Distributed by: (id)

zlytest=# select relfilenode, relname from pg_class where relname = 'idx_id';
 relfilenode | relname 
-------------+---------
       49317 | idx_id
(1 row)

#master上
[gpadmin@m1 gpseg-1]$ ll -h base/16384/49317
-rw------- 1 gpadmin gpadmin 32K Feb 20 14:48 base/16384/49317

#segment上
[root@seg1 gpseg0]# ll -h ../gpseg1/base/16384/49317
-rw------- 1 gpadmin gpadmin 64K Feb 20 14:49 ../gpseg1/base/16384/49317
```