# 3.4 更换 OSD Journal

本篇中部分内容来自 [zphj1987 —— 如何替换 Ceph 的 Journal](http://www.zphj1987.com/2016/07/26/如何替换Ceph的Journal/)

Ceph 在一块单独的磁盘上部署 OSD 的时候，是默认把 journal 和 OSD 放在同一块磁盘的不同分区上。有时候，我们可能需要把 OSD 的 journal 分区从一个磁盘替换到另一个磁盘上去。那么应该怎样替换 Ceph 的 journal 分区呢？

有两种方法来修改 Ceph 的 journal：

- 创建一个 journal 分区，在上面创建一个新的 journal。
- 转移已经存在的 journal 分区到新的分区上，这个适合整盘替换。

> Ceph 的 journal 是基于事务的日志，所以正确的下刷 journal 数据，然后重新创建 journal 并不会引起数据丢失，因为在下刷 journal 的数据的时候，osd 是停止的，一旦数据下刷后，这个 journal 是不会再有新的脏数据进来的。

### 第一种方法

1、首先给 Ceph 集群设置 `noout` 标志。

	root@mon:~# ceph osd set noout
	set noout

2、假设我们现在想要替换 osd.0 的 journal。首先查看 osd.0 当前的 journal 位置，当前使用的是 `/dev/sdb2` 分区。

	root@mon:~# ceph-disk list | grep osd.0
 	 /dev/sdb1 ceph data, active, cluster ceph, osd.0, journal /dev/sdb2

	root@mon:~# ll /var/lib/ceph/osd/ceph-0/journal
	lrwxrwxrwx 1 root root 58 May 24 15:06 /var/lib/ceph/osd/ceph-0/journal -> /dev/disk/by-partuuid/8e95b09d-ffa9-4163-b24c-b78020022797
	root@mon:~# ls -l /dev/disk/by-partuuid/
	total 0
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 39e9ad34-d7aa-4dec-865e-08952aa8aab5 -> ../../sdc1
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 8e95b09d-ffa9-4163-b24c-b78020022797 -> ../../sdb2
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 aaeca5fa-456a-4f45-8a8b-9de0c2642f44 -> ../../sdc2
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 d30a6d4a-6da4-4a81-a9e5-4bc69ebeec8f -> ../../sdb1

3、停止 osd.0 进程。

	stop ceph-osd id=0

4、下刷 journal 到 osd，使用 `-i` 指定需要替换 journal 的 osd 的编号。

	root@mon:~# ceph-osd -i 0 --flush-journal
	SG_IO: bad/missing sense data, sb[]:  70 00 05 00 00 00 00 0a 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
	SG_IO: bad/missing sense data, sb[]:  70 00 05 00 00 00 00 0a 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
	2016-11-08 13:17:58.355025 7f8351a72800 -1 flushed journal /var/lib/ceph/osd/ceph-0/journal for object store /var/lib/ceph/osd/ceph-0

5、删除旧的 journal 。

	root@mon:~# ll /var/lib/ceph/osd/ceph-0/journal
	lrwxrwxrwx 1 root root 58 May 24 15:06 /var/lib/ceph/osd/ceph-0/journal -> /dev/disk/by-partuuid/8e95b09d-ffa9-4163-b24c-b78020022797
	root@mon:~# rm -rf /var/lib/ceph/osd/ceph-0/journal

6、下面用 `/dev/sdc2` 分区重建 osd.0 的 journal 。查看 `/dev/sdc2` 的 `uuid`：

	root@mon:~# ls -l /dev/disk/by-partuuid/
	total 0
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 39e9ad34-d7aa-4dec-865e-08952aa8aab5 -> ../../sdc1
	lrwxrwxrwx 1 root root 10 Nov  8 13:17 8e95b09d-ffa9-4163-b24c-b78020022797 -> ../../sdb2
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 aaeca5fa-456a-4f45-8a8b-9de0c2642f44 -> ../../sdc2
	lrwxrwxrwx 1 root root 10 Nov  8 09:21 d30a6d4a-6da4-4a81-a9e5-4bc69ebeec8f -> ../../sdb1

新的 journal 的 uuid 的路径为 `/dev/disk/by-partuuid/aaeca5fa-456a-4f45-8a8b-9de0c2642f44` 。

7、新建 journal 的链接和 journal_uuid 文件：

	root@mon:~# ln -s /dev/disk/by-partuuid/aaeca5fa-456a-4f45-8a8b-9de0c2642f44 /var/lib/ceph/osd/ceph-0/journal
	root@mon:~# echo aaeca5fa-456a-4f45-8a8b-9de0c2642f44 > /var/lib/ceph/osd/ceph-0/journal_uuid

8、给 osd.0 创建 journal，使用 `-i` 指定 osd 的编号 。

	root@mon:~# ceph-osd -i 0 --mkjournal
	SG_IO: bad/missing sense data, sb[]:  70 00 05 00 00 00 00 0a 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
	2016-11-08 13:29:36.115461 7f64ec851800 -1 created new journal /var/lib/ceph/osd/ceph-0/journal for object store /var/lib/ceph/osd/ceph-0

9、查看新 journal 。

	root@mon:~# ceph-disk list | grep osd.0
 	 /dev/sdb1 ceph data, active, cluster ceph, osd.0, journal /dev/sdc2

10、启动 osd.0 。

	start ceph-osd id=0

11、去除 noout 的标记。

	ceph osd unset noout

12、检查集群的状态。

	root@mon:~# ceph -s
    cluster 614e77b4-c997-490a-a3f9-e89aa0274da3
     health HEALTH_OK
     monmap e5: 1 mons at {osd1=10.95.2.43:6789/0}
            election epoch 796, quorum 0 osd1
     osdmap e1067: 3 osds: 3 up, 3 in
            flags sortbitwise
      pgmap v309733: 384 pgs, 6 pools, 1148 MB data, 311 objects
            3597 MB used, 73162 MB / 76759 MB avail
                 384 active+clean

	root@mon:~# ceph osd tree
	ID WEIGHT  TYPE NAME                                     UP/DOWN REWEIGHT PRIMARY-AFFINITY                               
	-4 0.05997 root default                                                                    
	-1 0.01999     host mon                                                                    
	 0 0.01999         osd.0                                      up  1.00000          1.00000 
	-2 0.01999     host osd0                                                                   
	 1 0.01999         osd.1                                      up  1.00000          1.00000 
	-3 0.01999     host osd1                                                                   
	 2 0.01999         osd.2                                      up  1.00000          1.00000

### 第二种方法

这个属于备份和转移分区表的方法。

1、首先按方法一中的第 1 ~ 4 步，设置 `noout` 标志，停进程，下刷 journal。

2、备份需要替换 journal 的分区表。

	root@lab8106 ~# sgdisk --backup=/tmp/backup_journal_sdd /dev/sdd

3、 还原分区表。

	root@lab8106 ~# sgdisk --load-backup=/tmp/backup_journal_sde /dev/sde
	root@lab8106 ~# parted -s /dev/sde print

新的 journal 磁盘现在跟老的 journal 的磁盘的分区表一样的了。这意味着新的分区的 UUID 和老的相同的。如果选择的是这种备份还原的方法，那么 journal 的那个软连接是不需要进行修改的，因为两个磁盘的 uuid 是一样的，所以需要注意将老的磁盘拔掉或者清理掉分区，以免冲突。

4、重建 journal 。

	root@lab8106 ~# ceph-osd -i 0 --mkjournal

5、启动进程。

	root@lab8106 ~# start ceph-osd id=0

6、去除 `noout` 的标记。

	root@lab8106 ~# ceph osd unset noout
