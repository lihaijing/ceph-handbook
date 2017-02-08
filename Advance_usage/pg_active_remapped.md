# 6. PG 卡在 active + remapped 状态

----------

### 问题现象

有时，我们的 Ceph 集群可能会出现 PG 长时间卡在 `active + remapped` 的状态。

	root@ceph1:~# ceph -s
    	cluster 5ccdcb2d-961d-4dcb-a9ed-e8034c56cf71
     	 health HEALTH_WARN 88 pgs stuck unclean
     	 monmap e2: 1 mons at {ceph1=192.168.56.102:6789/0}, election epoch 1, quorum 0 ceph1
     	 osdmap e71: 4 osds: 4 up, 3 in
      	  pgmap v442: 256 pgs, 4 pools, 285 MB data, 8 objects
            	690 MB used, 14636 MB / 15326 MB avail
                  	  88 active+remapped
                 	 168 active+clean

### 产生问题的原因

出现这种情况，一般是做了 osd 的 `reweight` 操作引起的，这是因为一般在做 `reweight` 的操作的时候，根据算法，这个上面的 pg 是会尽量分布在这个主机上的，而 `crush reweight` 不变的情况下，去修改 osd 的 `reweight` 的时候，可能算法上会出现无法映射的问题。

### 如何解决

1、直接做 `ceph osd crush reweigh` 的调整即可避免这个问题，这个 straw 算法里面还是有点小问题的，在调整某个因子的时候会引起整个因子的变动。

2、从 FIREFLY (CRUSH_TUNABLES3) 开始 CRUSH 里面增加了一个参数：

    chooseleaf_vary_r

是否递归的进行 chooseleaf 尝试，如果非 0 ，就递归的进行，这个基于 parent 已经做了多少次尝试。默认值是 0 ，但是常常找不到合适的 mapping 。在计算成本和正确性上来看最优值是 1 。对于已经有大量数据的集群来说，从 0 调整为 1 将会有大量数值的迁移，调整为 4 或者 5 的话，将会找到一个更有效的映射，可以减少数据的移动。

查看当前的值：

	root@ceph1:~# ceph osd crush show-tunables |grep chooseleaf_vary_r
    	"chooseleaf_vary_r": 0,

修改 `chooseleaf_vary_r` 的值。

Hammer 版本下这个参数默认为：

>     tunable chooseleaf_vary_r 0

修改 Crush Map 的方法请参考本手册第一部分 [9. 管理 Crushmap](./Operation/manage_crushmap.md) 。

或者，直接修改 `crush tunables` 的值为 `optimal` 。

    ceph osd crush tunables optimal