# 1. PG 和 PGP 的区别

----------

本篇内容来自 [zphj1987 —— Ceph 中 PG 和 PGP 的区别](http://www.zphj1987.com/2016/10/19/Ceph%E4%B8%ADPG%E5%92%8CPGP%E7%9A%84%E5%8C%BA%E5%88%AB/)

### 前言

首先来一段关于 PG 和 PGP 区别的英文解释：

> PG = Placement Group  
> PGP = Placement Group for Placement purpose  
> pg_num = number of placement groups mapped to an OSD  
> When pg_num is increased for any pool, every PG of this pool splits into half, but they all remain mapped to their parent OSD.  
> Until this time, Ceph does not start rebalancing. Now, when you increase the pgp_num value for the same pool, PGs start to migrate from the parent to some other OSD, and cluster rebalancing starts. This is how PGP plays an important role.  
> By Karan Singh

以上是来自邮件列表的 `Karan Singh` 对 PG 和 PGP 的相关解释，他也是 `Learning Ceph` 和 `Ceph Cookbook` 的作者，以上的解释没有问题，我们来看下具体在集群里面如何作用。

### 实践

环境准备，因为是测试环境，我只准备了两台机器，每台机器 4 个 OSD，所以做了一些参数的设置，让数据尽量散列：

    osd_crush_chooseleaf_type = 0

以上为修改的参数，这个是让我的环境故障域为 OSD 分组的。

##### 创建测试需要的存储池

我们初始情况只创建一个名为 testpool 包含 6 个 PG 的存储池：

	ceph osd pool create testpool 6 6
	pool 'testpool' created

我们看一下默认创建完了后的 PG 分布情况：

	ceph pg dump pgs|grep ^1|awk '{print $1,$2,$15}'
	dumped pgs in format plain
	1.1 0 [3,6,0]
	1.0 0 [7,0,6]
	1.3 0 [4,1,2]
	1.2 0 [7,4,1]
	1.5 0 [4,6,3]
	1.4 0 [3,0,4]

我们写入一些对象，因为我们关心的不仅是 PG 的变动，同样关心 PG 内对象有没有移动，所以需要准备一些测试数据，这个调用原生 `rados` 接口写最方便。

	rados -p testpool bench 20 write --no-cleanup

我们再来查询一次：

	ceph pg dump pgs|grep ^1|awk '{print $1,$2,$15}'
	dumped pgs in format plain
	1.1 75 [3,6,0]
	1.0 83 [7,0,6]
	1.3 144 [4,1,2]
	1.2 146 [7,4,1]
	1.5 86 [4,6,3]
	1.4 80 [3,0,4]

可以看到写入了一些数据，其中的第二列为这个 PG 当中的对象的数目，第三列为 PG 所在的 OSD。

##### 增加 PG 测试

我们来扩大 PG 再看看：

	ceph osd pool set testpool pg_num 12
	set pool 1 pg_num to 12

再次查询：

	ceph pg dump pgs|grep ^1|awk '{print $1,$2,$15}'
	dumped pgs in format plain
	1.1 37 [3,6,0]
	1.9 38 [3,6,0]
	1.0 41 [7,0,6]
	1.8 42 [7,0,6]
	1.3 48 [4,1,2]
	1.b 48 [4,1,2]
	1.7 48 [4,1,2]
	1.2 48 [7,4,1]
	1.6 49 [7,4,1]
	1.a 49 [7,4,1]
	1.5 86 [4,6,3]
	1.4 80 [3,0,4]

可以看到上面新加上的 PG 的分布还是基于老的分布组合，并没有出现新的 OSD 组合，因为我们当前的设置是 pgp_num 为 6,那么三个 OSD 的组合的个数就是 6 个，因为当前为 12 个 PG，分布只能从 6 种组合里面挑选，所以会有重复的组合。

根据上面的分布情况，可以确定的是，增加 PG 操作会引起 PG 内部对象分裂，分裂的份数是根据新增 PG 组合重复情况来确定的，比如上面的情况：

- `1.1` 的对象分成了两份 `[3,6,0]`
- `1.3` 的对象分成了三份 `[4,1,2]`
- `1.4` 的对象没有拆分 `[3,0,4]`

**结论： 增加 PG 会引起 PG 内的对象分裂，也就是在 OSD 上创建了新的 PG 目录，然后进行部分对象的 `move` 的操作。**

##### 增加 PGP 测试

我们将原来的 PGP 从 6 调整到 12：

	ceph osd pool set testpool pgp_num 12

	ceph pg dump pgs|grep ^1|awk '{print $1,$2,$15}'
	dumped pgs in format plain
	1.a 49 [1,2,6]
	1.b 48 [1,6,2]
	1.1 37 [3,6,0]
	1.0 41 [7,0,6]
	1.3 48 [4,1,2]
	1.2 48 [7,4,1]
	1.5 86 [4,6,3]
	1.4 80 [3,0,4]
	1.7 48 [1,6,0]
	1.6 49 [3,6,7]
	1.9 38 [1,4,2]
	1.8 42 [1,2,3]

可以看到 PG 里面的对象并没有发生变化，而 PG 所在的对应关系发生了变化。

我们看下与调整 PGP 前后的对比：

	*1.1 37 [3,6,0]          1.1 37 [3,6,0]*
	 1.9 38 [3,6,0]          1.9 38 [1,4,2]
	*1.0 41 [7,0,6]          1.0 41 [7,0,6]*
	 1.8 42 [7,0,6]          1.8 42 [1,2,3]
	*1.3 48 [4,1,2]          1.3 48 [4,1,2]*
	 1.b 48 [4,1,2]          1.b 48 [1,6,2]
	 1.7 48 [4,1,2]          1.7 48 [1,6,0]
	*1.2 48 [7,4,1]          1.2 48 [7,4,1]*
	 1.6 49 [7,4,1]          1.6 49 [3,6,7]
	 1.a 49 [7,4,1]          1.a 49 [1,2,6]
	*1.5 86 [4,6,3]          1.5 86 [4,6,3]*
	*1.4 80 [3,0,4]          1.4 80 [3,0,4]*

可以看到其中最原始的 6 个 PG 的分布并没有变化（标注了 `*` 号），变化的是后增加的 PG，也就是将重复的 PG 分布进行重新分布，这里并不是随机完全打散，而是根据需要去进行重分布。

**结论： 调整 PGP 不会引起 PG 内的对象的分裂，但是会引起 PG 的分布的变动。**

### 总结

- PG 是指定存储池存储对象的目录有多少个，PGP 是存储池 PG 的 OSD 分布组合个数。
- PG 的增加会引起 PG 内的数据进行分裂，分裂到相同的 OSD 上新生成的 PG 当中。
- PGP 的增加会引起部分 PG 的分布进行变化，但是不会引起 PG 内对象的变动。