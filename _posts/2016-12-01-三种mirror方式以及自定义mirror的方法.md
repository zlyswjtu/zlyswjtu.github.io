---
title: GP支持的Segment镜像方式及自定义方法
date: 2016-12-01 23:00:00
categories:
- Greenplum
tags:
- Greenplum
---

## GP支持三种mirror方式：

- groupped mirror（默认）
- spread mirror
- 自定义mirror

groupped方式，将一台segment host上的所有primary segment的镜像存放在另一台segment host上，因此两台物理机就可以互做镜像，采用这种方式segment host数目必须是2的倍数。

spread方式，要求一台segment host上的每一个primary segment的镜像必须存放到不同的segment host上，若一台segment host上的primary segment数目为n，则物理机segment host数目至少为n+1.

自定义mirror的方式，在系统初始化或扩容时groupped和spread方式都可以自动生成配置文件，而自定义mirror的方式需要手工生成配置文件。

## 大圈小圈问题：
groupped方式，相当于2个一圈，当两个中一个Segment host挂掉，由于mirror全在另一台上，另一台负载增加1倍

spread方式，相当于所有主机构成一个圈，若一台segment host上的primary segment数目为n，要求物理机segment host数目至少为n+1，当其中一台segment host挂掉时，其他segment host负载增加1/n

自定义mirror方式，可以指定3个一圈，4个一圈，...n个一圈（扩容必须是n的倍数？）
当一个segment host挂掉，由于mirror平均分布在同一圈的(n-1)个segment host上，同一圈每个segment host负载增加1/(n-1)

## 自定mirror
### 初始化
在gpinitsystem初始化时，不添加mirror（gpinitsystem只能将mirror初始化为groupped或spread），在gpinitsystem执行完成后，使用gpaddmirrors根据自定义配置添加mirror。

可以使用gpaddmirrors -o add_mirror_config_file得到一个配置文件，但是这个配置文件里是groupped方式组织mirror，修改该文件为自定义方式即可。例如我有3个segment host，每个segment host上有两个primary segment，其中gpseg0和gpseg1的primary在seg1.com上，gpseg2和gpseg3的的primary在seg2.com上，gpseg4和gpseg5的的primary在seg3.com上，运行gpaddmirrors命令得到的groupped配置文件内容个如下：
```
filespaceOrder=
mirror0=0:seg2.com:41000:42000:43000:/gpadmin/data/mirror/gpseg0
mirror1=1:seg2.com:41001:42001:43001:/gpadmin/data/mirror/gpseg1
mirror2=2:seg3.com:41000:42000:43000:/gpadmin/data/mirror/gpseg2
mirror3=3:seg3.com:41001:42001:43001:/gpadmin/data/mirror/gpseg3
mirror4=4:seg1.com:41000:42000:43000:/gpadmin/data/mirror/gpseg4
mirror5=5:seg3.com:41001:42001:43001:/gpadmin/data/mirror/gpseg5
```
配置文件采用的格式如下：
```
filespaceOrder=[filespace1_fsname[:filespace2_fsname:...]
mirror[content]=content:address:port:mir_replication_port:pri_replication_
port:fselocation[:fselocation:...]
```
可以从上述配置文件中看出，groupped方式的mirror分布是一台segment host上的所有primary的mirror都在另一台segment host上，例如seg1.com上的gpseg0和gpseg1，它们的mirror都放在seg2.com上。

我们希望mirror方式为3个一圈，即其中一台segment host上的primary segment要分布在另外两台segment host上，只要修改mirror存放的主机名即可，修改时注意同一台机器上的分配的端口号不能冲突。于是将配置文件修改如下：
```
filespaceOrder=
mirror0=0:seg2.com:41000:42000:43000:/gpadmin/data/mirror/gpseg0
mirror1=1:seg3.com:41001:42001:43001:/gpadmin/data/mirror/gpseg1
mirror2=2:seg3.com:41000:42000:43000:/gpadmin/data/mirror/gpseg2
mirror3=3:seg1.com:41001:42001:43001:/gpadmin/data/mirror/gpseg3
mirror4=4:seg1.com:41000:42000:43000:/gpadmin/data/mirror/gpseg4
mirror5=5:seg2.com:41001:42001:43001:/gpadmin/data/mirror/gpseg5
```
运行gpaddmirrors -i add_mirror_config_file即可完成mirror初始化。
