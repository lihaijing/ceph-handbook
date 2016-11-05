# 1.3 监控 OSD

某 OSD 的状态可以是在集群内（ `in` ）或集群外（ `out` ）、也可以是运行着的（ `up` ）或不在运行的（ `down` ）。如果一个 OSD 处于 `up` 状态，它也可以是在集群之内 `in` （你可以读写数据）或者之外 `out` 。如果它以前是 `in` 但最近 `out` 了， Ceph 会把 PG 迁移到其他 OSD 上。如果某个 OSD `out` 了， CRUSH 就不会再分配 PG 给它。如果它 `down` 了，其状态也应该是 `out` 。默认在 OSD `down` 掉 300s 后会标记它为 `out` 状态。

**注意：如果某个 OSD 状态为 down & in ，必定有问题，而且集群处于非健康状态。**

OSD 监控的一个重要事情就是，当集群启动并运行时，所有 OSD 也应该是启动（ `up` ）并在集群内（ `in` ）运行的。用下列命令查看：

    ceph osd stat

其结果会告诉你 osd map 的版本（ eNNNN ），总共有多少个 OSD 、几个是 `up` 的、几个是 `in` 的。

    osdmap e26753: 3 osds: 2 up, 3 in

如果处于 `in` 状态的 OSD 多于 `up` 的，用下列命令看看哪些 `ceph-osd` 守护进程没在运行：

	ceph osd tree ::

	ID WEIGHT  TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY 
	-1 0.05997 root default                                      
	-2 0.01999     host ceph01                                   
	 0 0.01999         osd.0        up  1.00000          1.00000 
	-3 0.01999     host ceph02                                   
	 1 0.01999         osd.1        up  1.00000          1.00000 
	-4 0.01999     host ceph03                                   
	 2 0.01999         osd.2      down  1.00000          1.00000

如果有 OSD 处于 down 状态，请尝试启动该 OSD，启动命令见[1.1 操作集群](./operate_cluster.md)。如果启动失败，请参考[2.2 常见 OSD 故障处理](../Troubleshooting/troubleshooting_osd.md)中的相关部分进行处理。
