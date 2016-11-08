# 3.2 Monitor 的备份和恢复

本篇内容来自 [徐小胖'blog —— monitor 的增删改备](http://xuxiaopang.com/2016/10/26/exp-monitor-operation/) 

### Monitor 的备份

每个 MON 的数据都是保存在数据库内的，这个数据库位于 `/var/lib/ceph/mon/$cluster-$hostname/store.db` ，这里的 `$cluster` 是集群的名字， `$hostname` 为主机名，MON 的所有数据即目录 `/var/lib/ceph/mon/$cluster-$hostname/` ，备份好这个目录之后，就可以在任一主机上恢复 MON 了。

这里参考了下[这篇文章](https://blog.widodh.nl/2014/03/safely-backing-up-your-ceph-monitors/)里面的备份方法，简单讲基本思路就是，停止一个 MON，然后将这个 MON 的数据库压缩保存到其他路径，再开启 MON。文中提到了之所以要停止 MON 是要保证 levelDB 数据库的完整性。然后可以做个定时任务一天或者一周备份一次。

另外最好把 `/etc/ceph/` 目录也备份一下。

这个备份路径最好是放到其他节点上，不要保存到本地，因为一般 MON 节点要坏就坏一台机器。

这里给出文中提到的备份方法：

    service ceph stop mon
    tar czf /var/backups/ceph-mon-backup_$(date +'%a').tar.gz /var/lib/ceph/mon
    service ceph start mon
    #for safety, copy it to other nodes
    scp /var/backups/* someNode:/backup/

### Monitor 的恢复

现在有一个 Ceph 集群，包含 3 个 monitors： ceph-1 、ceph-2 和 ceph-3 。

	[root@ceph-1 cluster]# ceph -s
    	cluster 844daf70-cdbc-4954-b6c5-f460d25072e0
     	 health HEALTH_OK
     	 monmap e2: 3 mons at {ceph-1=192.168.56.101:6789/0,ceph-2=192.168.56.102:6789/0,ceph-3=192.168.56.103:6789/0}
            	election epoch 8, quorum 0,1,2 ceph-1,ceph-2,ceph-3
     	 osdmap e13: 3 osds: 3 up, 3 in
      	  pgmap v20: 64 pgs, 1 pools, 0 bytes data, 0 objects
            	101 MB used, 6125 GB / 6125 GB avail
                  	  64 active+clean

假设发生了某种故障，导致这 3 台 MON 节点全都无法启动，这时 Ceph 集群也将变得不可用。我们可以通过前面备份的数据库文件来恢复 MON。当某个集群的所有的 MON 节点都挂掉之后，我们可以将最新的备份的数据库解压到其他任意一个节点上，新建 monmap，注入，启动 MON，推送 config，重启 OSD就好了。

将 `ceph-1` 的 `/var/lib/ceph/mon/ceph-ceph-1/` 目录的文件拷贝到新节点 `ceph-4` 的 `/var/lib/ceph/mon/ceph-ceph-4/` 目录下（或者从备份路径拷贝到 `ceph-4` 节点），一定要注意目录的名称！这里 `ceph-1` 的 IP 为 `172.23.0.101`， `ceph-4` 的 IP 为 `192.168.56.104` 。`ceph-4` 节点为一个只安装了 ceph 程序的干净节点。注意下面指令执行的节点。

	[root@ceph-1 ~]# ip a |grep 172
    	inet 172.23.0.101/24 brd 172.23.0.255 scope global enp0s8
	[root@ceph-1 ~]# ping ceph-4
	PING ceph-4 (192.168.56.104) 56(84) bytes of data.
	64 bytes from ceph-4 (192.168.56.104): icmp_seq=1 ttl=63 time=0.463 ms

	[root@ceph-4 ~]# mkdir /var/lib/ceph/mon/ceph-ceph-4
	[root@ceph-1 ~]# scp -r /var/lib/ceph/mon/ceph-ceph-1/*  ceph-4:/var/lib/ceph/mon/ceph-ceph-4/
	done                                                             100%    0     0.0KB/s   00:00    
	keyring                                                          100%   77     0.1KB/s   00:00    
	LOCK                                                             100%    0     0.0KB/s   00:00    
	LOG                                                              100%   21KB  20.6KB/s   00:00    
	161556.ldb                                                       100% 2098KB   2.1MB/s   00:00    
    ......
	MANIFEST-161585                                                  100%  709     0.7KB/s   00:00    
	CURRENT                                                          100%   16     0.0KB/s   00:00    
	sysvinit                                                         100%    0     0.0KB/s   00:00    

同时，将 `/etc/ceph` 目录文件也拷贝到 `ceph-4` 节点，然后将 `ceph.conf` 中的 `mon_initial_members` 修改为 `ceph-4`。

	[root@ceph-1 ~]# scp /etc/ceph/* ceph-4:/etc/ceph/
	ceph.client.admin.keyring                                                         100%   63     0.1KB/s   00:00    
	ceph.conf                                                                         100%  236     0.2KB/s   00:00  
  
	[root@ceph-4 ~]# vim /etc/ceph/ceph.conf 
	[root@ceph-4 ~]# cat /etc/ceph/ceph.conf 
	[global]
	fsid = 844daf70-cdbc-4954-b6c5-f460d25072e0
	mon_initial_members = ceph-4
	mon_host = 192.168.56.104

新建一个 monmap，使用原来集群的 `fsid`，并且将 `ceph-4` 加入到这个 monmap，然后将 monmap 注入到 `ceph-4` 的 MON 数据库中，最后启动 `ceph-4` 上的 MON 进程。

	[root@ceph-4 ~]# monmaptool --create --fsid 844daf70-cdbc-4954-b6c5-f460d25072e0 --add ceph-4 192.168.56.104 /tmp/monmap 
	[root@ceph-4 ~]# ceph-mon -i ceph-4 --inject-monmap /tmp/monmap 
	[root@ceph-4 ~]# ceph-mon -i ceph-4
	[root@ceph-4 ~]# ceph -s
    	cluster 844daf70-cdbc-4954-b6c5-f460d25072e0
     	 health HEALTH_ERR
            	64 pgs are stuck inactive for more than 300 seconds
            	64 pgs degraded
            	64 pgs stuck inactive
            	64 pgs undersized
     	 monmap e6: 1 mons at {ceph-4=192.168.56.104:6789/0}
            	election epoch 13, quorum 0 ceph-4
     	 osdmap e36: 3 osds: 1 up, 1 in
      	  pgmap v58: 64 pgs, 1 pools, 0 bytes data, 0 objects
            	34296 kB used, 2041 GB / 2041 GB avail
                  	  64 undersized+degraded+peered

好消息是，`ceph -s` 有了正确的输出，坏消息就是 `HEALTH_ERR` 了。不过不用担心，这时候将 `ceph-4` 的 `ceph.conf` 推送到其他所有节点上，再重启 OSD 集群就可以恢复正常了。