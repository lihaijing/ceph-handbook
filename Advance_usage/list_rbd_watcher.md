# 10. 查看使用 RBD 镜像的客户端

----------

有时候删除 rbd image 会提示当前 rbd image 正在使用中，无法删除：

    rbd rm foo
    2016-11-09 20:16:14.018332 7f81877a07c0 -1 librbd: image has watchers - not removing
    Removing image: 0% complete...failed.
    rbd: error: image still has watchers
    This means the image is still open or the client using it crashed. Try again after closing/unmapping it or waiting 30s for the crashed client to timeout.

所以希望能查看到底是谁在使用 rbd image。

对于 rbd image 的使用，目前主要有两种形式：内核模块 map 后再 mount ；通过 libvirt 等供给虚拟机使用。都是利用 `rados listwatchers` 去查看，只是两种形式下需要读取的文件不一样。

### 内核模块 map

查看方法如下：

	# 查看 rbd pool 中的 image
	root@ceph1:~# rbd ls
	foo
	ltest
	test

	# 查看 foo 的使用者
	root@ceph1:~# rados -p rbd listwatchers foo.rbd
	watcher=10.202.0.82:0/1760414582 client.1332795 cookie=1

	# 去对应主机上检查
	root@ceph2:~#rbd showmapped |grep foo
	0  rbd  foo   -    /dev/rbd0 

### 虚拟机通过 libvirt 使用 rbd image

查看方法如下：

	# 查看该虚拟机卷的信息
	root@ceph1:~# rbd info volumes/volume-ee0c4077-a607-4bc9-a8cf-e893837361f3
	rbd image 'volume-ee0c4077-a607-4bc9-a8cf-e893837361f3':
	        size 1024 MB in 256 objects
	        order 22 (4096 kB objects)
	        block_name_prefix: rbd_data.13c3277850f538
	        format: 2
	        features: layering, striping
	        flags: 
	        parent: images/3601459f-060b-460f-b73b-db74237d922e@snap
	        overlap: 40162 kB
	        stripe unit: 4096 kB
	        stripe count: 1

	# 查看该 image 的使用者
	root@ceph1:~# rados -p volumes listwatchers rbd_header.13c3277850f538
	watcher=10.202.0.21:0/1043073 client.1298745 cookie=140256077850496

我们登录控制节点，查看对应的 cinder 卷：

	root@controller:~# cinder list | grep ee0c4077-a607-4bc9-a8cf-e893837361f3
	| ee0c4077-a607-4bc9-a8cf-e893837361f3 | in-use |      |  1   |      -      |   true   | 96ee1aad-af27-4c9d-968f-291dbb2766a1 |

该卷挂载在 ID 为 `96ee1aad-af27-4c9d-968f-291dbb2766a1` 的虚拟机上。通过 `nova show` 命令验证该虚拟机是否在物理机 10.202.0.21 ( computer21 )上：
	
	root@controller:~# nova show 96ee1aad-af27-4c9d-968f-291dbb2766a1
	+--------------------------------------+---------------------------------------------------------------------------------+
	| Property                             | Value                                                                           |
	+--------------------------------------+---------------------------------------------------------------------------------+
	| OS-DCF:diskConfig                    | AUTO                                                                            |
	| OS-EXT-AZ:availability_zone          | az_ip                                                                           |
	| OS-EXT-SRV-ATTR:host                 | computer21                                                                      |
	| OS-EXT-SRV-ATTR:hostname             | byp-volume                                                                      |
	| OS-EXT-SRV-ATTR:hypervisor_hostname  | computer21                                                                      |
	| OS-EXT-SRV-ATTR:instance_name        | instance-00000989                                                               |
	| OS-EXT-SRV-ATTR:kernel_id            |                                                                                 |
	| OS-EXT-SRV-ATTR:launch_index         | 0                                                                               |
	| OS-EXT-SRV-ATTR:ramdisk_id           |                                                                                 |
	| OS-EXT-SRV-ATTR:reservation_id       | r-hwiyx15c                                                                      |
	| OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                                        |
	| OS-EXT-SRV-ATTR:user_data            | -                                                                               |
	| OS-EXT-STS:power_state               | 1                                                                               |
	| OS-EXT-STS:task_state                | -                                                                               |
	| OS-EXT-STS:vm_state                  | active                                                                          |
	| OS-SRV-USG:launched_at               | 2016-11-09T08:19:41.000000                                                      |
	| OS-SRV-USG:terminated_at             | -                                                                               |
	| accessIPv4                           |                                                                                 |
	| accessIPv6                           |                                                                                 |
	| blue-net network                     | 192.168.50.27                                                                   |
	| config_drive                         |                                                                                 |
	| created                              | 2016-11-09T07:53:59Z                                                            |
	| description                          | byp-volume                                                                      |
	| flavor                               | m1.small (2)                                                                    |
	| hostId                               | 5104ee1a0538048d6ef80b14563a0cbac461f86523c5c81f5d18069e                        |
	| host_status                          | UP                                                                              |
	| id                                   | 96ee1aad-af27-4c9d-968f-291dbb2766a1                                            |
	| image                                | Attempt to boot from volume - no image supplied                                 |
	| key_name                             | octavia_ssh_key                                                                 |
	| locked                               | False                                                                           |
	| metadata                             | {}                                                                              |
	| name                                 | byp-volume                                                                      |
	| os-extended-volumes:volumes_attached | [{"id": "ee0c4077-a607-4bc9-a8cf-e893837361f3", "delete_on_termination": true}] |
	| progress                             | 0                                                                               |
	| security_groups                      | default                                                                         |
	| status                               | ACTIVE                                                                          |
	| tenant_id                            | f21a9c86d7114bf99c711f4874d80474                                                |
	| updated                              | 2016-11-09T08:19:41Z                                                            |
	| user_id                              | 142d8663efce464c89811c63e45bd82e                                                |
	| zq-test network                      | 192.168.6.6                                                                     |
	+--------------------------------------+---------------------------------------------------------------------------------+
	