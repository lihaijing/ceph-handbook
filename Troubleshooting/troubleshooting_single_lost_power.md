# 2.5 单个Ceph节点宕机处理

在某些情况下，如服务器硬件故障，造成单台Ceph节点宕机无法启动，可以按照本节所示流程将该节点上的OSD移除集群，从而达到Ceph集群的恢复。

### 2.5.1 单台Ceph节点宕机处理步骤

1. 登陆ceph monitor节点，查询ceph状态：

   `ceph health detail`

2. 将故障节点上的所有osd设置成out，该步骤会触发数据recovery,需要等待数据迁移完成, 同时观察虚拟机是否正常：

   `ceph osd out osd_id`

3. 从crushmap将osd移除，该步骤会触发数据reblance，等待数据迁移完成，同时观察虚拟机是否正常：

   `ceph osd crush remove osd_name`

4. 删除osd节点的认证:`ceph auth del osd_name`

5. 删除osd节点:`ceph osd rm osd_id`


### 2.5.2 恢复后检查步骤

1. 检查ceph集群状态正常
2. 检查虚拟机状态正常
3. 楚天云人员检查虚拟机业务是否正常
4. 检查平台服务正常：nova、cinder、glance
5. 创建新卷正常
6. 创建虚拟机正常



