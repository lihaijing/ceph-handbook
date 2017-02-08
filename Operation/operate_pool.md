# 8. 操作 Pool

----------

如果你开始部署集群时没有创建存储池， Ceph 会用默认存储池 rbd 存放数据。存储池提供的功能：

- **自恢复力：** 你可以设置在不丢数据的前提下允许多少 OSD 失效。对多副本存储池来说，此值是一对象应达到的副本数。典型配置是存储一个对象和它的一个副本（即 size = 2 ），但你可以更改副本数；对纠删编码的存储池来说，此值是编码块数（即纠删码配置里的 m = 2 ）。
- **归置组：** 你可以设置一个存储池的 PG 数量。典型配置给每个 OSD 分配大约 100 个 PG，这样，不用过多计算资源就能得到较优的均衡。配置了多个存储池时，要考虑到这些存储池和整个集群的 PG 数量要合理。
- **CRUSH 规则：** 当你在存储池里存数据的时候，与此存储池相关联的 CRUSH 规则集可控制 CRUSH 算法，并以此操纵集群内对象及其副本的复制（或纠删码编码的存储池里的数据块）。你可以自定义存储池的 CRUSH 规则。
- **快照：** 用 `ceph osd pool mksnap` 创建快照的时候，实际上创建了某一特定存储池的快照。

要把数据组织到存储池里，你可以列出、创建、删除存储池，也可以查看每个存储池的使用统计数据。

### 8.1 列出存储池
要列出集群的存储池，命令如下：

	ceph osd lspools

在新安装好的集群上，默认只有一个 rbd 存储池。

### 8.2 创建存储池

创建存储池前可以先看看存储池、PG 和 CRUSH 配置参考。你最好在配置文件里重置默认 PG 数量，因为默认值并不理想。

例如：

	osd pool default pg num = 100
	osd pool default pgp num = 100

要创建一个存储池，执行：

	ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated] \
        	[crush-ruleset-name] [expected-num-objects]
	ceph osd pool create {pool-name} {pg-num}  {pgp-num}   erasure \
        	[erasure-code-profile] [crush-ruleset-name] [expected_num_objects]

各参数含义如下：

- `{pool-name}`：存储池名称，必须唯一。
- `{pg_num}`： 存储池的 PG 数目。
- `{pgp_num}`： 存储池的 PGP 数目，此值应该和 PG 数目相等。
- `{replicated|erasure}`：存储池类型，可以是副本池（保存多份对象副本，以便从丢失的 OSD 恢复）或纠删池（获得类似 RAID5 的功能）。多副本存储池需更多原始存储空间，但已实现所有 Ceph 操作；纠删存储池所需原始存储空间较少，但目前仅实现了部分 Ceph 操作。
- `[crush-ruleset-name]`：此存储池所用的 CRUSH 规则集名字。指定的规则集必须存在。对于多副本（** replicated** ）存储池来说，其默认规则集由 `osd pool default crush replicated ruleset` 配置决定，此规则集必须存在。 对于用 `erasure-code` 编码的纠删码（ **erasure** ）存储池来说，不同的 `{pool-name}` 所使用的默认（ `default` ）纠删码配置是不同的，如果它不存在的话，会显式地创建它。
- `[erasure-code-profile=profile]`：仅用于**纠删**存储池。指定纠删码配置文件，此配置必须已由 `osd erasure-code-profile set` 定义。
- `[expected-num-objects]`：为这个存储池预估的对象数。设置此值（要同时把 **filestore merge threshold** 设置为负数）后，在创建存储池时就会拆分 PG 文件夹，以免运行时拆分文件夹导致延时增大。

关于如何计算合适的 `pg_num` 值，可以使用 Ceph 官方提供的一个计算工具 [**pgcalc**](http://ceph.com/pgcalc/) 。

### 8.3 设置存储池配额

存储池配额可设置最大字节数、和/或每个存储池最大对象数。

	ceph osd pool set-quota {pool-name} [max_objects {obj-count}] [max_bytes {bytes}]

例如：

	ceph osd pool set-quota data max_objects 10000

要取消配额，设置为 `0` 即可。

### 8.4 删除存储池

要删除一存储池，执行：

	ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]

如果你给自建的存储池创建了定制的规则集，你不需要存储池时最好也删除规则集。

	ceph osd pool get {pool-name} crush_ruleset

加入规则集是 “123”，可以这样选择存储池：

	ceph osd dump | grep "^pool" | grep "crush_ruleset 123"

如果你曾严格地创建了用户及其权限给一个存储池，但存储池已不存在，最好也删除那些用户。

    ceph auth list | grep -C 5 {pool-name}
    ceph auth del {user}

### 8.5 重命名存储池

要重命名一个存储池，执行：

	ceph osd pool rename {current-pool-name} {new-pool-name}

如果重命名了一个存储池，且认证用户对每个存储池都有访问权限，那你必须用新存储池名字更新用户的能力（即 caps ）。

### 8.6 查看存储池统计信息

要查看某存储池的使用统计信息，执行命令：

	rados df

### 8.7 给存储池做快照

要给某存储池做快照，执行命令：

	ceph osd pool mksnap {pool-name} {snap-name}

### 8.8 删除存储池的快照

要删除某存储池的一个快照，执行命令：

	ceph osd pool rmsnap {pool-name} {snap-name}

### 8.9 获取存储池选项值

要获取一个存储池的选项值，执行命令：

	ceph osd pool get {pool-name} {key}

### 8.10 调整存储池选项值

要设置一个存储池的选项值，执行命令：

	ceph osd pool set {pool-name} {key} {value}

常用选项介绍：

- `size`：设置存储池中的对象副本数，详情参见设置对象副本数。仅适用于副本存储池。
- `min_size`：设置 I/O 需要的最小副本数，详情参见设置对象副本数。仅适用于副本存储池。
- `pg_num`：计算数据分布时的有效 PG 数。只能大于当前 PG 数。
- `pgp_num`：计算数据分布时使用的有效 PGP 数量。小于等于存储池的 PG 数。
- `crush_ruleset`：
- `hashpspool`：给指定存储池设置/取消 HASHPSPOOL 标志。
- `target_max_bytes`：达到 `max_bytes` 阀值时会触发 Ceph 冲洗或驱逐对象。
- `target_max_objects`：达到 `max_objects` 阀值时会触发 Ceph 冲洗或驱逐对象。
- `scrub_min_interval`：在负载低时，洗刷存储池的最小间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_min_interval 。
- `scrub_max_interval`：不管集群负载如何，都要洗刷存储池的最大间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_max_interval 。
- `deep_scrub_interval`：“深度”洗刷存储池的间隔秒数。如果是 0 ，就按照配置文件里的 osd_deep_scrub_interval 。

### 8.11 设置对象副本数

要设置多副本存储池的对象副本数，执行命令：

	ceph osd pool set {poolname} size {num-replicas}

**重要：** `{num-replicas}` 包括对象自身，如果你想要对象自身及其两份拷贝共计三份，指定 size 为 3 。

例如：

	ceph osd pool set data size 3

你可以在每个存储池上执行这个命令。注意，一个处于降级模式的对象，其副本数小于 `pool size` ，但仍可接受 I/O 请求。为保证 I/O 正常，可用 `min_size` 选项为其设置个最低副本数。例如：

	ceph osd pool set data min_size 2

这确保数据存储池里任何副本数小于 `min_size` 的对象都不会收到 I/O 了。

### 8.12 获取对象副本数

要获取对象副本数，执行命令：

	ceph osd dump | grep 'replicated size'

Ceph 会列出存储池，且高亮 `replicated size` 属性。默认情况下， Ceph 会创建一对象的两个副本（一共三个副本，或 size 值为 3 ）。