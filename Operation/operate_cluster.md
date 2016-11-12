# 1.1 操作集群

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