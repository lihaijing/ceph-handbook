# 1.11 修改集群配置

启动 Ceph 存储集群时，各守护进程都从同一个配置文件（即默认的 `ceph.conf` ）里查找它自己的配置。`ceph.conf` 中可配置参数很多，有时我们需要根据实际环境对某些参数进行修改。

修改的方式分为两种：直接修改 `ceph.conf` 配置文件中的参数值，修改完后需要重启 Ceph 进程才能生效。或在运行中动态地进行参数调整，无需重启进程。

### 查看运行时配置

如果你的 Ceph 存储集群在运行，而你想看一个在运行进程的配置，用下面的命令：

	ceph daemon {daemon-type}.{id} config show | less

如果你现在位于 osd.0 所在的主机，命令将是：

	ceph daemon osd.0 config show | less

### 修改配置文件

Ceph 配置文件可用于配置存储集群内的所有守护进程、或者某一类型的所有守护进程。要配置一系列守护进程，这些配置必须位于能收到配置的段落之下，比如：

**`[global]`**

描述： `[global]` 下的配置影响 Ceph 集群里的所有守护进程。  
实例： `auth supported = cephx`

**`[osd]`**

描述： `[osd]` 下的配置影响存储集群里的所有 `ceph-osd` 进程，并且会覆盖 `[global]` 下的同一选项。  
实例： `osd journal size = 1000`

**`[mon]`**

描述： `[mon]` 下的配置影响集群里的所有 `ceph-mon` 进程，并且会覆盖 `[global]` 下的同一选项。  
实例： `mon addr = 10.0.0.101:6789`

**`[mds]`**

描述： `[mds]` 下的配置影响集群里的所有 `ceph-mds` 进程，并且会覆盖 `[global]` 下的同一选项。  
实例： `host = myserver01`

**`[client]`**

描述： `[client]` 下的配置影响所有客户端（如挂载的 Ceph 文件系统、挂载的块设备等等）。  
实例： `log file = /var/log/ceph/radosgw.log`

全局设置影响集群内所有守护进程的例程，所以 `[global]` 可用于设置适用所有守护进程的选项。但可以用这些覆盖 `[global]` 设置：

1. 在 `[osd]` 、 `[mon]` 、 `[mds]` 下更改某一类进程的配置。
2. 更改特定进程的设置，如 `[osd.1]` 。

覆盖全局设置会影响所有子进程，明确剔除的例外。

### 运行中动态调整

Ceph 可以在运行时更改 `ceph-osd` 、 `ceph-mon` 、 `ceph-mds` 守护进程的配置，此功能在增加/降低日志输出、启用/禁用调试设置、甚至是运行时优化的时候非常有用。Ceph 集群提供两种方式的调整，使用 `tell` 的方式和 `daemon` 设置的方式。

##### tell 方式设置

下面是使用 tell 命令的修改方法：

	ceph tell {daemon-type}.{id or *} injectargs --{name} {value} [--{name} {value}]

用 `osd` 、 `mon` 、 `mds` 中的一个替代 `{daemon-type}` ，你可以用星号（ `*` ）更改一类进程的所有实例配置、或者更改某一具体进程 ID （即数字或字母）的配置。例如提高名为 `osd.0` 的 `ceph-osd` 进程之调试级别的命令如下：

	ceph tell osd.0 injectargs --debug-osd 20 --debug-ms 1

在 `ceph.conf` 文件里配置时用空格分隔关键词；但在命令行使用的时候要用下划线或连字符（ `_` 或 `-` ）分隔，例如 `debug osd` 变成 `debug-osd` 。

##### daemon 方式设置

除了上面的 tell 的方式调整，还可以使用 daemon 的方式进行设置。

1、获取当前的参数

	ceph daemon osd.1 config get mon_osd_full_ratio

	{
	"mon_osd_full_ratio": "0.98"
	}

2、修改配置

	ceph daemon osd.1 config set mon_osd_full_ratio 0.97

	{
	"success": "mon_osd_full_ratio = '0.97' "
	}


3、检查配置

	ceph daemon osd.1 config get mon_osd_full_ratio

	{
	"mon_osd_full_ratio": "0.97"
	}


**注意：** 重启进程后配置会恢复到默认参数，在进行在线调整后，如果这个参数是后续是需要使用的，那么就需要将相关的参数写入到配置文件 `ceph.conf` 当中。

##### 两种设置方式的使用场景
使用 tell 的方式适合对整个集群进行设置，使用 `*` 号进行匹配，就可以对整个集群的角色进行设置。而出现节点异常无法设置时候，只会在命令行当中进行报错，不太便于查找。

使用 daemon 进行设置的方式就是一个个的去设置，这样可以比较好的反馈，此方法是需要在设置的角色所在的主机上进行设置。