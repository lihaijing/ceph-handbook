# 3.7 查看 RBD 镜像的位置

有时，我们需要查看某个 RBD 镜像的对象都存放在哪些 PG 中，这些 PG 又分布在哪些 OSD 上。可以利用下面的 shell 脚本来实现快速查看 RBD 镜像的位置。

	#!/bin/bash

	# USAGE:./rbd-loc <pool> <image>

	if [ -z ${1} ] || [ -z ${2} ];
	then
    	echo "USAGE: ./rbd-loc <pool> <image>"
	exit 1
	fi

	rbd_prefix=$(rbd -p ${1} info ${2} | grep block_name_prefix | awk '{print $2}')
	for i in $(rados -p ${1} ls | grep ${rbd_prefix})
	do
    	ceph osd map ${1} ${i}
	done

执行的效果如下所示：

	root@mon:~# rbd ls -p images
	fc5b017d-fc74-4a59-80bb-a5a76e26dd4e

	root@mon:~# ./rbd-loc.sh images fc5b017d-fc74-4a59-80bb-a5a76e26dd4e
	osdmap e1078 pool 'images' (9) object 'rbd_data.1349f035c101d9.0000000000000001' -> pg 9.99b52d94 (9.14) -> up ([1,2,0], p1) acting ([1,2,0], p1)
	osdmap e1078 pool 'images' (9) object 'rbd_data.1349f035c101d9.0000000000000002' -> pg 9.40973ca2 (9.22) -> up ([0,2,1], p0) acting ([0,2,1], p0)
	osdmap e1078 pool 'images' (9) object 'rbd_data.1349f035c101d9.0000000000000003' -> pg 9.86758b2c (9.2c) -> up ([1,2,0], p1) acting ([1,2,0], p1)
	osdmap e1078 pool 'images' (9) object 'rbd_data.1349f035c101d9.0000000000000004' -> pg 9.3c8e78f6 (9.36) -> up ([0,1,2], p0) acting ([0,1,2], p0)
	osdmap e1078 pool 'images' (9) object 'rbd_data.1349f035c101d9.0000000000000000' -> pg 9.ffc971ff (9.3f) -> up ([0,2,1], p0) acting ([0,2,1], p0)

该测试环境只有 3 个 host， 每个 host 上 1 个 OSD ，3 副本设置。