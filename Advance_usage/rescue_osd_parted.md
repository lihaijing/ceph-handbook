# 3.5 清空 OSD 的分区表后如何恢复

本篇内容来自 [zphj1987 —— 不小心清空了 Ceph 的 OSD 的分区表如何恢复](http://www.zphj1987.com/2016/09/24/%E4%B8%8D%E5%B0%8F%E5%BF%83%E6%B8%85%E7%A9%BA%E4%BA%86Ceph%E7%9A%84OSD%E7%9A%84%E5%88%86%E5%8C%BA%E8%A1%A8%E5%A6%82%E4%BD%95%E6%81%A2%E5%A4%8D/)

假设不小心对 Ceph OSD 执行了 `ceph-deploy disk zap` 这个操作，那么该 OSD 对应磁盘的分区表就丢失了。本文讲述了在这种情况下如何进行恢复。

### 破坏环境

我们现在有一个正常的集群，假设用的是默认的分区的方式，我们先来看看默认的分区方式是怎样的。

1、查看默认的分区方式。

	root@mon:~# ceph-disk  list
	···
	/dev/sdb :
 	 /dev/sdb1 ceph data, active, cluster ceph, osd.0, journal /dev/sdb2
 	 /dev/sdb2 ceph journal, for /dev/sdb1
	···

2、查看分区情况

	root@mon:~# parted -s /dev/sdb print
	Model: SEAGATE ST3300657SS (scsi)
	Disk /dev/sdb: 300GB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	Disk Flags: 

	Number  Start   End     Size    File system  Name          Flags
 	 2      1049kB  1074MB  1073MB               ceph journal
 	 1      1075MB  300GB   299GB   xfs          ceph data

3、破坏 `/dev/sdb` 的分区表，该磁盘对应的是 `osd.0` 。

	root@mon:~/ceph# ceph-deploy disk zap mon:/dev/sdb
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (1.5.34): /usr/bin/ceph-deploy disk zap mon:/dev/sdb
	···
	[mon][DEBUG ] Warning: The kernel is still using the old partition table.
	[mon][DEBUG ] The new table will be used at the next reboot.
	[mon][DEBUG ] GPT data structures destroyed! You may now partition the disk using fdisk or
	[mon][DEBUG ] other utilities.
	···

即使这个 osd 在被使用，还是被破坏了，这里假设上面的就是一个误操作，我们看下带来了哪些变化：

	root@mon:~/ceph# ll /var/lib/ceph/osd/ceph-0/journal
	lrwxrwxrwx 1 root root 58 Sep 24 00:02 /var/lib/ceph/osd/ceph-0/journal -> /dev/disk/by-partuuid/bd81471d-13ff-44ce-8a33-92a8df9e8eee

如果你用命令行看，就可以看到上面的链接已经变红了，分区没有了：

	root@mon:~/ceph# ceph-disk  list 
	/dev/sdb :
	 /dev/sdb1 other, xfs, mounted on /var/lib/ceph/osd/ceph-0
	 /dev/sdb2 other

已经跟上面有变化了，没有 ceph 的相关分区信息了：

    root@mon:~/ceph# parted -s /dev/sdb print
    Model: SEAGATE ST3300657SS (scsi)
    Disk /dev/sdb: 300GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start  End  Size  File system  Name  Flags

分区表完全没有信息了，到这我们可以确定分区表完全没了，如果现在重启将会发生什么？重启以后这个磁盘就是一个裸盘，没有分区的裸盘，所以此时千万**不能重启**！

### 恢复环境

首先一个办法就是当这个 OSD 坏了，然后直接按照删除节点，添加节点的方法去处理，这个应该是最主流、最通用的处理办法，但是这个方法在生产环境当中引发的数据迁移还是非常大的。我们尝试做恢复，这就是本篇主要讲的东西。

1、首先设置 `noout` 标志。

	root@mon:~/ceph# ceph osd set noout

2、停止 OSD 。

	root@mon:~/ceph# stop ceph-osd id=0

现在的 OSD 还是有进程的，所以需要停止掉再做处理。

3、查看其他 OSD 的分区信息（这里要求磁盘一致）。

	root@mon:~/ceph# parted -s /dev/sdc unit s print
	Model: SEAGATE ST3300657SS (scsi)
	Disk /dev/sdc: 585937500s
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	Disk Flags: 

	Number  Start     End         Size        File system  Name          Flags
	 2      2048s     2097152s    2095105s                 ceph journal
	 1      2099200s  585937466s  583838267s  xfs          ceph data

记住上面的数值， `print` 的时候是加了 `unit s` 这个是要精确的值的，下面的步骤会用到的这些数值。

4、进行分区表的恢复。

	root@mon:~/ceph# parted -s /dev/sdb mkpart ceph_data 2099200s 585937466s
	root@mon:~/ceph# parted -s /dev/sdb mkpart ceph_journal 2048s 2097152s

5、再次检查 /dev/sdb 的分区表。

	root@mon:~/ceph# parted -s /dev/sdb print
	Model: SEAGATE ST3300657SS (scsi)
	Disk /dev/sdb: 300GB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	Disk Flags: 

	Number  Start   End     Size    File system  Name          Flags
	 2      1049kB  1074MB  1073MB               ceph_journal
	 1      1075MB  300GB   299GB   xfs          ceph_data

可以看到，分区表已经回来了。

6、重新挂载分区。

    root@mon:~/ceph# umount /var/lib/ceph/osd/ceph-0
    root@mon:~/ceph# partprobe
    root@mon:~/ceph# mount /dev/sdb1 /var/lib/ceph/osd/ceph-0

7、删除旧的 journal ，重建 osd.0 的 journal。 

	root@mon:~/ceph# rm -rf /var/lib/ceph/osd/ceph-0/journal

	root@mon:~/ceph# ceph-osd -i 0 --osd-journal=/dev/sdb2 --mkjournal
	SG_IO: bad/missing sense data, sb[]:  70 00 05 00 00 00 00 0a 00 00 00 00 20 00 01 cf 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
	2016-09-24 00:36:06.595992 7f9d0afbc880 -1 created new journal /dev/sdb2 for object store /var/lib/ceph/osd/ceph-0
	root@mon:~/ceph# ln -s /dev/sdb2 /var/lib/ceph/osd/ceph-0/journal
	root@mon:~/ceph# ll /var/lib/ceph/osd/ceph-0/journal
	lrwxrwxrwx 1 root root 9 Sep 24 00:37 journal -> /dev/sdb2

注意上面的操作 `--osd-journal=/dev/sdb2` 这个地方，此处写成 `/dev/sdb2` 是便于识别，这个地方在实际操作中要写上 `/dev/sdb2` 的 uuid 的路径。

	root@mon:~/ceph# ll /dev/disk/by-partuuid/03fc6039-ad80-4b8d-86ec-aeee14fb3bb6 
	lrwxrwxrwx 1 root root 10 Sep 24 00:33 /dev/disk/by-partuuid/03fc6039-ad80-4b8d-86ec-aeee14fb3bb6 -> ../../sdb2

也就是这个链接的一串内容，这是为了防止盘符串了的情况下无法找到 journal 的问题。

8、启动 OSD 。

	root@mon:~/ceph# start ceph-osd id=0

检查下，到这 osd.0 就成功地恢复了。
