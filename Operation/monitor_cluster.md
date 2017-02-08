# 2. 监控集群

----------

集群运行起来后，你可以用 `ceph` 工具来监控集群的状态，典型的监控项目包括检查 OSD 状态、monitor 的状态、PG 的状态和元数据服务器的状态（目前楚天云环境并没有部署元数据服务器）。

### 2.1 交互模式

要在交互模式下运行 `ceph` ，不要带参数运行 `ceph` ，例如：

    ceph
    ceph> health
    ceph> status
    ceph> quorum_status
    ceph> mon_status
    
### 2.2 检查集群的监控状况
启动集群后、读写数据前，先检查下集群的健康状态。你可以用下面的命令检查：

	ceph health

如果你的配置文件或 keyring 文件不在默认路径下，你得在命令中指定：

	ceph -c /path/to/conf -k /path/to/keyring health

集群刚起来的时候，你也许会碰到像 `HEALTH_WARN XXX num placement groups stale` 这样的健康告警，等一会再检查下。集群准备好的话 `ceph health` 会给出 `HEALTH_OK` 这样的消息，这时候就可以开始使用集群了。

### 2.3 观察集群
要观察集群内正发生的事件，打开一个新终端，然后输入：

	ceph -w

Ceph 会打印各种事件。例如一个包括 3 个 Mon、和 33 个 OSD 的 Ceph 集群可能会打印出这些：

	  cluster b84b887e-9e0c-4211-8423-e0596939cd36
       health HEALTH_OK
       monmap e1: 3 mons at {OPS-ceph1=192.168.219.30:6789/0,OPS-ceph2=192.168.219.31:6789/0,OPS-ceph3=192.168.219.32:6789/0}
              election epoch 94, quorum 0,1,2 OPS-ceph1,OPS-ceph2,OPS-ceph3
       osdmap e1196: 33 osds: 33 up, 33 in
        pgmap v1789894: 2752 pgs, 7 pools, 590 GB data, 110 kobjects
              1154 GB used, 83564 GB / 84719 GB avail
                  2752 active+clean
	client io 0 B/s rd, 25852 B/s wr, 7 op/s

	2016-11-04 20:20:13.682953 mon.0 [INF] pgmap v1789893: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 44908 B/s wr, 14 op/s
	2016-11-04 20:20:15.686275 mon.0 [INF] pgmap v1789894: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 25852 B/s wr, 7 op/s
	2016-11-04 20:20:16.690680 mon.0 [INF] pgmap v1789895: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 32345 B/s wr, 16 op/s
	2016-11-04 20:20:17.694259 mon.0 [INF] pgmap v1789896: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 57170 B/s wr, 32 op/s
	2016-11-04 20:20:18.698200 mon.0 [INF] pgmap v1789897: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 33148 B/s wr, 16 op/s
	2016-11-04 20:20:20.701697 mon.0 [INF] pgmap v1789898: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 16333 B/s wr, 5 op/s
	2016-11-04 20:20:21.705719 mon.0 [INF] pgmap v1789899: 2752 pgs: 2752 active+clean; 590 GB data, 1154 GB used, 83564 GB / 84719 GB avail; 0 B/s rd, 17705 B/s wr, 12 op/s

输出信息里包含：

- 集群的 ID
- 集群健康状况
- monitor map 版本和 mon 法定人数状态
- OSD map 版本和 OSD 状态摘要
- PG map 版本
- PG 和 Pool 的数量
- 集群存储的数据量，对象的总量，以及集群的已用容量/总容量/可用容量
- 客户端的 iops 信息

### 2.4 检查集群的使用情况
要检查集群的数据用量及其在存储池内的分布情况，可以用 `df` 选项，它和 Linux 上的 `df` 相似。如下：

	ceph df

得到的输出信息大致如下：

    GLOBAL:
        SIZE       AVAIL      RAW USED     %RAW USED 
        84719G     83564G        1154G          1.36 
	POOLS:
        NAME                  ID     USED       %USED     MAX AVAIL     OBJECTS 
    	rbd                   0           0         0        41381G           0 
    	volumes               1        284G      0.34        41381G       57904 
    	images                2        224G      0.27        41381G       39024 
    	backups               3           0         0        41381G           1 
    	vms                   4      28736M      0.03        41381G        4325 
    	volumes-ssd           5      53758M      0.06        41381G       11854 
    	fitos_backup_pool     7       1286M         0        41381G         354 

输出的 **GLOBAL** 段展示了数据所占用集群存储空间的概要。

- **SIZE：** 集群的总容量。
- **AVAIL：** 集群的可用空间总量。
- **RAW USED：**已用存储空间总量。
- **% RAW USED：**已用存储空间比率。用此值对比 `full ratio` 和 `near full ratio` 来确保不会用尽集群空间。

输出的 **POOLS** 段展示了存储池列表及各存储池的大致使用率。本段没有反映出副本、克隆和快照的占用情况。例如，如果你把 1MB 的数据存储为对象，理论使用率将是 1MB ，但考虑到副本数、克隆数、和快照数，实际使用量可能是 2MB 或更多。

- **NAME：**存储池名字。
- **ID：**存储池唯一标识符。
- **USED：**大概数据量，单位为 KB 、MB 或 GB ；
- **%USED：**各存储池的大概使用率。
- **Objects：**各存储池内的大概对象数。

注意： **POOLS** 段内的数字是估计值，它们不包含副本、快照或克隆。因此，各 Pool 的 **USED** 和 **%USED** 数量之和不会达到 **GLOBAL** 段中的 **RAW USED** 和 **%RAW USED** 数量。

### 2.5 检查集群状态
要检查集群的状态，执行下面的命令：

    ceph status

或者：

    ceph -s

在交互模式下，输入 `status` 然后按回车：

    ceph> status

Ceph 将打印集群状态，例如一个包括 1 个监视器、和 2 个 OSD 的小型 Ceph 集群可能打印：

    cluster b370a29d-9287-4ca3-ab57-3d824f65e339
     health HEALTH_OK
     monmap e1: 1 mons at {ceph1=10.0.0.8:6789/0}, election epoch 2, quorum 0 ceph1
     osdmap e63: 2 osds: 2 up, 2 in
      pgmap v41332: 952 pgs, 20 pools, 17130 MB data, 2199 objects
        	115 GB used, 167 GB / 297 GB avail
                   1 active+clean+scrubbing+deep
             	 951 active+clean

### 2.6 检查 OSD 状态
你可以执行下列命令来确定 OSD 状态为 `up` 且 `in` ：

    ceph osd stat

或者：

    ceph osd dump

你也可以根据 OSD 在 CRUSH MAP 里的位置来查看：

    ceph osd tree

Ceph 会打印 CRUSH 树，包括 host 的名称、它上面的 OSD 例程、状态及权重：

	ID WEIGHT  TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY
    -1 0.05997 root default
    -2 0.01999     host ceph01
     0 0.01999         osd.0        up  1.00000          1.00000
    -3 0.01999     host ceph02
     1 0.01999         osd.1        up  1.00000          1.00000
    -4 0.01999     host ceph03
     2 0.01999         osd.2        up  1.00000          1.00000

另请参考本手册第一部分 [3. 监控 OSD](./monitor_osd.md)。

### 2.7 检查 Mon 状态

如果集群中有多个 Mon（很可能），你启动集群后、读写数据前应该检查 Mon 法定人数状态。运行着多个 Mon 时必须形成法定人数，最好周期性地检查 Mon 状态来确定它们在运行。

要查看 Mon map，执行下面的命令：

    ceph mon stat

或者：

    ceph mon dump

要检查监视器的法定人数状态，执行下面的命令：

    ceph quorum_status

Ceph 会返回法定人数状态，例如，包含 3 个监视器的 Ceph 集群可能返回下面的：

    { "election_epoch": 94,
      "quorum": [
            0,
            1,
            2],
      "quorum_names": [
            "OPS-ceph1",
            "OPS-ceph2",
            "OPS-ceph3"],
      "quorum_leader_name": "OPS-ceph1",
      "monmap": { "epoch": 1,
          "fsid": "b84b887e-9e0c-4211-8423-e0596939cd36",
	  "modified": "2016-11-04 20:19:57.333655",
	  "created": "2016-06-23 14:53:07.171558",
	  "mons": [
			{"rank": 0,
			 "name": "OPS-ceph1",
			 "addr": "192.168.219.30:6789\/0"
		    {"rank": 1,
			 "name": "OPS-ceph2",
			 "addr": "192.168.219.31:6789\/0"},
			{"rank": 2,
			 "name": "OPS-ceph3",
			 "addr":"192.168.219.32:6789\/0"}
           ]
       }
    }

### 2.8 检查 MDS 状态

元数据服务器为 Ceph 文件系统提供元数据服务，不过楚天云当前并未部署 MDS 。元数据服务器有两种状态： `up | down` 和 `active | inactive` ，执行下面的命令查看元数据服务器状态为 `up` 且 `active` ：

    ceph mds stat

要展示元数据集群的详细状态，执行下面的命令：

    ceph mds dump

### 2.9 检查 PG 状态

PG 把对象映射到 OSD 。监控 PG 时，我们希望它们的状态是 `active` 且 `clean` 。详情请参考本手册第一部分 [4. 监控 PG](./monitor_pg.md)

### 2.10 使用管理套接字

Ceph 管理套接字允许你通过套接字接口查询守护进程，它们默认存在于 `/var/run/ceph` 下。要通过管理套接字访问某个守护进程，先登录它所在的主机、再执行下列命令：

    ceph daemon {daemon-name}
    ceph daemon {path-to-socket-file}

比如，这是下面这两种用法是等价的：

    ceph daemon osd.0 foo
    ceph daemon /var/run/ceph/ceph-osd.0.asok foo

用下列命令查看可用的管理套接字命令：

    ceph daemon {daemon-name} help

管理套接字命令允许你在运行时查看和修改配置。

另外，你可以在运行时直接修改配置选项（也就是说管理套接字会绕过 Mon，不要求你直接登录宿主主机，不像 `ceph {daemon-type} tell {id} injectargs` 会依赖监视器）。