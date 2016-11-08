# 3.3 修改 Cinder/Glance 进程的最大可用 FD

本篇内容是根据生产环境中遇到的实际问题进行的总结。

### 背景

在生产环境中遇到这样一个问题：  
下发删除卷消息无法成功删除卷，后通过 cinder 命令行命令 `cinder service-list` 查询cinder 服务状态，发现 `cinder-volume host : r2202002controller@rbd-sata` 服务状态为 `DOWN` 。

	|  cinder-volume   |  r2202002controller@rbd-sata   | nova | enabled  |    down  |

该状态表明该 cinder-volume 进程已经没有正常上报心跳，处于无法处理请求的状态。

### 原因分析

通过现场复现问题和查看 cinder-volume 日志，发现在出问题的时间点都有删除卷的操作下发，但是有的卷一直未结束删除卷流程，直到重启 cinder-volume 进程时才恢复。

	2016-09-23 13:42:48.176 44907 INFO cinder.volume.manager [req-0182ed51-a4a7-44e4-bd3e-3d456610a135 ff01ec05ed4442799ebad096c1aa2921 584c5f2fed764ec9b840319eb2cd0608 - - -] volume b38ea39e-b1f5-4af6-a7b1-40fe1d5e80ee: deleting

	2016-09-23 16:52:08.290 52145 INFO cinder.volume.manager [req-be3fe7fc-39fe-4bc3-9a70-d5be1e7330ce - - - - -] Resuming delete on volume: b38ea39e-b1f5-4af6-a7b1-40fe1d5e80ee

怀疑在删除 RDB 卷流程有挂死问题，通过进一步查看日志，发现最后走到调用 RDB Client 删除卷时就中断了:

	2016-09-23 13:43:27.911 44907 DEBUG cinder.volume.drivers.rbd [req-0182ed51-a4a7-44e4-bd3e-3d456610a135 ff01ec05ed4442799ebad096c1aa2921 584c5f2fed764ec9b840319eb2cd0608 - - -] deleting rbd volume volume-b38ea39e-b1f5-4af6-a7b1-40fe1d5e80  delete_volume /usr/local/lib/python2.7/dist-packages/cinder/volume/drivers/rbd.py:666

因为该问题发生在 Ceph 集群扩容后，因此进一步怀疑和之前 Nova 挂卷后读取变慢问题一样是由于进程打开文件过多引起异常，后在实验环境中通过打开 RDB client 日志，重启 cinder-volume 进程。

    [client]
    rbd cache = true
    rbd cache writethrough until flush = true
    log file = /var/log/ceph/ceph.client.log
    debug client = 20
    debug rbd = 20
    debug librbd = 20
    debug objectcacher = 20

然后增加 cinder-volume 进程删卷时的打开文件数成功复现该问题。RDB client 日志中报 `too many open files` 异常，后在楚天云环境上通过复现问题和打开 RDB Client 日志观察到同样的异常，因此可以确定是由于扩容后 OSD 节点增多，RDB client 删除卷时需要和所有 OSD 建立 socket 链接，这样就会超过目前环境中 cinder-volume 进程允许打开的文件数，导致异常发生，进入挂死状态。

	2016-09-26 20:08:48.953810 7f099566b700 -1 -- 192.168.219.2:0/43062437 >> 192.168.219.130:6812/12006 pipe(0x7f0acdcdc090 sd=-1 :0 s=1 pgs=0 cs=0 l=1 c=0x7f0acdcae7e0).connect couldn't created socket (24) Too many open files
	2016-09-26 20:08:48.953803 7f099556a700 -1 -- 192.168.219.2:0/43062437 >> 192.168.219.113:6802/2740 pipe(0x7f0acdce0330 sd=-1 :0 s=1 pgs=0 cs=0 l=1 c=0x7f0acdcb2210).connect couldn't created socket (24) Too many open files
	2016-09-26 20:08:48.953898 7f0995267700 -1 -- 192.168.219.2:0/43062437 >> 192.168.219.179:6812/31845 pipe(0x7f0acdcf12f0 sd=-1 :0 s=1 pgs=0 cs=0 l=1 c=0x7f0acdced990).connect couldn't created socket (24) Too many open files
	2016-09-26 20:08:48.953913 7f099596e700 -1 -- 192.168.219.2:0/43062437 >> 192.168.219.169:6804/8418 pipe(0x7f0acdccd2f0 sd=-1 :0 s=1 pgs=0 cs=0 l=1 c=0x7f0acdcc0a90).connect couldn't created socket (24) Too many open files
	2016-09-26 20:08:48.953932 7f0995368700 -1 -- 192.168.219.2:0/43062437 >> 192.168.219.136:6806/23096 pipe(0x7f0acdce8f40 sd=-1 :0 s=1 pgs=0 cs=0 l=1 c=0x7f0acdcc7cf0).connect couldn't created socket (24) Too many open files

### 解决方案

由于问题原因是才 cinder-volume 允许打开的文件数没有随着 Ceph 集群的扩容做相应的调整，因此解决方案是要调整 cinder-volume 进程的允许打开文件数，目前调整为 `65535`（根据测试，发现每个删卷请求要建立大约 1000 多个链接，因此调整为 `65535` 后可以支持并发删除约 60 个 RDB 卷，后续版本会考虑基于性能基线，进行接口并发操作数量的限制，防止无限制的并发删卷导致文件打开数过大）。

1、cinder-volume 进程被封装成了 Ubuntu 上的 upstart 任务。修改 cinder-volume 进程启动配置文件 `/etc/init/cinder-volume.conf` ，增加一行配置：

    limit nofile 65535 65535

`limit` 变量用来设置任务的资源限制。

	root@R1controller1:~# vi /etc/init/cinder-volume.conf
	description "Cinder Volume"

	start on (local-filesystems and net-device-up IFACE!=lo)
	stop on runlevel [016]

	respawn
    limit nofile 65535 65535

	exec su -s /bin/sh -C "exec cinder-volume --config-file /etc/cinder/cinder.conf > /dev/null 2>&1" root

2、重启 cinder-volume 进程，并用 `cat proc/pid/limits` 查询设置是否生效。

	root@R1controller1:~# service cinder-volume restart
	cinder-volume stop/waiting
	cinder-volume start/running, process 6298

	root@R1controller1:~# cat /etc/6298/limits | grep open
	Max open files       65535               65535               files

3、检查 cinder-volume 服务是否为 `UP` 状态。

	root@R1controller1:~# cinder service-list
	+------------------+---------------------------+------+---------+-------+----------------------------+-----------------+
	|      Binary      |            Host           | Zone |  Status | State |         Updated_at         | Disabled Reason |
	+------------------+---------------------------+------+---------+-------+----------------------------+-----------------+
	|  cinder-backup   |      OPS-controller1      | nova | enabled |   up  | 2016-09-07T20:38:30.000000 |       None      |
	|  cinder-backup   |      OPS-controller2      | nova | enabled |   up  | 2016-09-07T20:38:26.000000 |       None      |
	| cinder-scheduler |      OPS-controller1      | nova | enabled |   up  | 2016-09-07T20:38:29.000000 |       None      |
	| cinder-scheduler |      OPS-controller2      | nova | enabled |   up  | 2016-09-07T20:38:29.000000 |       None      |
	|  cinder-volume   | OPS-controller1@netapp_fc | nova | enabled |   up  | 2016-09-07T20:38:32.000000 |       None      |
	|  cinder-volume   |  OPS-controller1@rbd-sas  | nova | enabled |   up  | 2016-09-07T20:38:33.000000 |       None      |
	|  cinder-volume   |  OPS-controller1@rbd-ssd  | nova | enabled |   up  | 2016-09-07T20:38:33.000000 |       None      |
	|  cinder-volume   |  OPS-controller2@rbd-sas  | nova | enabled |   up  | 2016-09-07T20:38:25.000000 |       None      |
	|  cinder-volume   |  OPS-controller2@rbd-ssd  | nova | enabled |   up  | 2016-09-07T20:38:27.000000 |       None      |
	+------------------+---------------------------+------+---------+-------+----------------------------+-----------------+

4、重新进行删除卷操作，卷可以被正常删除。

5、因为 Glance 后端也对接的是 Ceph 集群，为防止 Glance 也出现该问题，建议也修改 Glance 进程的启动配置文件 `/etc/init/glance-api.conf` ，增加 `limit nofile 65535 65535` 配置行。

6、重启 glance-api 进程。

	service glance-api restart

7、重启完成后上传测试镜像，测试镜像可以正常上传和删除。
