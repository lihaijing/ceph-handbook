# 10. 修改 MON IP

----------

Ceph 客户端和其他 Ceph 守护进程通过 `ceph.conf` 来发现 monitor。但是 monitor 之间是通过 mon map 而非 `ceph.conf` 来发现彼此。

### 10.1 修改 MON IP（正确的方法）

仅修改 `ceoh.conf` 中 mon 的 IP 是不足以确保集群中的其他 monitor 收到更新的。要修改一个 mon 的 IP，你必须先新增一个使用新 IP 的 monitor（参考[1.6 增加/删除 Monitor](./add_rm_mon.md)），确保这个新 mon 成功加入集群并形成法定人数。然后，删除使用旧 IP 的 mon。最后，更新 `ceph.conf` ，以便客户端和其他守护进程可以知道新 mon 的 IP。

比如，假设现有 3 个 monitors：

	[mon.a]
        	host = host01
        	addr = 10.0.0.1:6789
	[mon.b]
        	host = host02
        	addr = 10.0.0.2:6789
	[mon.c]
        	host = host03
        	addr = 10.0.0.3:6789

把 `mon.c` 变更为 `mon.d` 。按照本手册第一部分 [6. 增加/删除 Monitor](./add_rm_mon.md)，增加一个 `mon.d` ，host 设为 `host04`，IP 地址设为 `10.0.0.4`。先启动 `mon.d` ，再按照 [6. 增加/删除 Monitor](./add_rm_mon.md) 删除 `mon.c` ，否则会破坏法定人数。

### 10.2 修改 MON IP（不太友好的方法）

有时，monitor 需要迁移到一个新的网络中、数据中心的其他位置或另一个数据中心。这时，需要为集群中所有的 monitors 生成一个新的 mon map （指定了新的 MON IP），再注入每一个 monitor 中。

还以前面的 mon 配置为例。假定想把 monitor 从 `10.0.0.x` 网段改为 `10.1.0.x` 网段，这两个网段直接是不通的。执行下列步骤：

1、获取 mon map。

	ceph mon getmap -o {tmp}/{filename}

2、下面的例子说明了 monmap 的内容。

	$ monmaptool --print {tmp}/{filename}

    monmaptool: monmap file {tmp}/{filename}
    epoch 1
    fsid 224e376d-c5fe-4504-96bb-ea6332a19e61
    last_changed 2012-12-17 02:46:41.591248
    created 2012-12-17 02:46:41.591248
    0: 10.0.0.1:6789/0 mon.a
    1: 10.0.0.2:6789/0 mon.b
    2: 10.0.0.3:6789/0 mon.c

3、删除已有的 monitors。

	$ monmaptool --rm a --rm b --rm c {tmp}/{filename}

	monmaptool: monmap file {tmp}/{filename}
	monmaptool: removing a
	monmaptool: removing b
	monmaptool: removing c
	monmaptool: writing epoch 1 to {tmp}/{filename} (0 monitors)

4、新增 monitor。

	$ monmaptool --add a 10.1.0.1:6789 --add b 10.1.0.2:6789 --add c 10.1.0.3:6789 {tmp}/{filename}

	monmaptool: monmap file {tmp}/{filename}
	monmaptool: writing epoch 1 to {tmp}/{filename} (3 monitors)

5、检查 monmap 的新内容。

    $ monmaptool --print {tmp}/{filename}
    
    monmaptool: monmap file {tmp}/{filename}
    epoch 1
    fsid 224e376d-c5fe-4504-96bb-ea6332a19e61
    last_changed 2012-12-17 02:46:41.591248
    created 2012-12-17 02:46:41.591248
    0: 10.1.0.1:6789/0 mon.a
    1: 10.1.0.2:6789/0 mon.b
    2: 10.1.0.3:6789/0 mon.c

此时，我们假定 monitor 已在新位置安装完毕。下面的步骤就是分发新的 monmap 并注入到各新 monitor 中。

1、停止所有的 monitor 。必须停止 mon 守护进程才能进行 monmap 注入。

2、注入 monmap。

	ceph-mon -i {mon-id} --inject-monmap {tmp}/{filename}

3、重启各 monitors 。