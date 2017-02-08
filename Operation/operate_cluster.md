# 1. 操作集群 
------------

## 1.1 用 UPSTART 控制 CEPH

用 ceph-deploy 把 Ceph Cuttlefish 及更高版部署到 Ubuntu 14.04 上，你可以用基于事件的 Upstart 来启动、关闭 Ceph 节点上的守护进程。 Upstart 不要求你在配置文件里定义守护进程例程。

### 1.1.1 列出节点上所有的 Ceph 作业和实例

	sudo initctl list | grep ceph

### 1.1.2 启动所有守护进程

要启动某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo stop ceph-all

### 1.1.3 停止所有守护进程

要停止某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo stop ceph-all

### 1.1.4 按类型启动所有守护进程

要启动某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo start ceph-osd-all
	sudo start ceph-mon-all
	sudo start ceph-mds-all

### 1.1.5 按类型停止所有守护进程

要停止某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo stop ceph-osd-all
	sudo stop ceph-mon-all
	sudo stop ceph-mds-all

### 1.1.6 启动单个进程

要启动某节点上一个特定的守护进程例程，用下列命令之一：

    sudo start ceph-osd id={id}
    sudo start ceph-mon id={hostname}
    sudo start ceph-mds id={hostname}

例如：

    sudo start ceph-osd id=1
    sudo start ceph-mon id=ceph-server
    sudo start ceph-mds id=ceph-server

### 1.1.7 停止单个进程

要停止某节点上一个特定的守护进程例程，用下列命令之一：

    sudo stop ceph-osd id={id}
    sudo stop ceph-mon id={hostname}
    sudo stop ceph-mds id={hostname}

例如：

    sudo stop ceph-osd id=1
    sudo stop ceph-mon id=ceph-server
    sudo stop ceph-mds id=ceph-server

## 1.2 用 SYSTEMD 控制 CEPH

对于所有支持 `systemd` 的 Linux 发行版（CentOS 7, Fedora, Debian Jessie 8.x, SUSE），使用原生的 `systemd` 文件来代替传统的 `sysvinit` 脚本。不过需要注意，这和 Ceph 的版本也有关系。如果 CentOS 7 + Jewel，使用的就是 `systemd` 。

### 1.2.1 列出节点上所有的 Ceph systemd units

	sudo systemctl status ceph\*.service ceph\*.target

### 1.2.2 启动所有守护进程

要启动某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl start ceph.target

### 1.2.3 停止所有守护进程

要停止某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl stop ceph\*.service ceph\*.target

### 1.2.4 按类型启动所有守护进程

要启动某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl start ceph-osd.target
	sudo systemctl start ceph-mon.target
	sudo systemctl start ceph-mds.target

### 1.2.5 按类型停止所有守护进程

要停止某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl stop ceph-mon\*.service ceph-mon.target
	sudo systemctl stop ceph-osd\*.service ceph-osd.target
	sudo systemctl stop ceph-mds\*.service ceph-mds.target

### 1.2.6 启动单个进程

要启动某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl start ceph-osd@{id}
	sudo systemctl start ceph-mon@{hostname}
	sudo systemctl start ceph-mds@{hostname}


### 1.2.7 停止单个进程

要停止某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl stop ceph-osd@{id}
	sudo systemctl stop ceph-mon@{hostname}
	sudo systemctl stop ceph-mds@{hostname}


## 1.3 把 CEPH 当服务运行

在某些环境下，还可以把 Ceph 当做服务来运行，比如 CentOS 7 + Hammer 。

### 1.3.1 启动所有守护进程

要启动本节点上的所有 Ceph 守护进程，用下列命令：

	sudo service ceph [start|restart]

### 1.3.2 停止所有守护进程

要停止本节点上的所有 Ceph 守护进程，用下列命令：

	sudo service ceph stop

### 1.3.3 按类型启动所有守护进程

要启动本节点上的某一类 Ceph 守护进程，用下列命令：

	sudo service ceph start {daemon-type}

### 1.3.4 按类型停止所有守护进程

要停止本节点上的某一类 Ceph 守护进程，用下列命令：

	sudo service ceph stop {daemon-type}

### 1.3.5 启动单个进程

要启动本节点上某个特定的守护进程例程，用下列命令：

	sudo service ceph start {daemon-type}.{instance}

### 1.3.6 停止单个进程

要停止本节点上某个特定的守护进程例程，用下列命令：

	sudo service ceph start {daemon-type}.{instance}
