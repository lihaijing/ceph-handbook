# 3.8 查看 RBD 镜像的实际大小

本篇内容来自 [zphj1987 —— 如何统计 Ceph 的 RBD 真实使用容量](http://www.zphj1987.com/2016/09/08/%E5%A6%82%E4%BD%95%E7%BB%9F%E8%AE%A1Ceph%E7%9A%84RBD%E7%9C%9F%E5%AE%9E%E4%BD%BF%E7%94%A8%E5%AE%B9%E9%87%8F/)

Ceph 的 rbd 一直有个问题就是无法清楚的知道这个分配的空间里面到底使用了多少，使用 `rbd info` 命令查询出来的容量是预分配的总容量而非实际使用容量。在 Jewel 版中提供了一个新的接口去查询，对于老版本来说可能同样有这个需求，本篇将详细介绍如何解决这个问题。

目前已知的有三种查询方法：

1. 使用 `rbd du` 查询（Jewel 版才支持）
2. 使用 `rbd diff`
3. 根据对象统计的方法进行统计

### 方法一：使用 rbd du 查询

此命令是 Jewel 版本才支持，此处暂不做讨论。

### 方法二：使用 rbd diff

	root@mon:~# rbd diff rbd/mysql-img | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
	52.8047 MB

### 方法三：根据对象统计的方法进行统计

在集群非常大的时候，再按上面的方法一个个查询，需要花很长的时间，并且需要时不时的跟集群进行交互。方法三是把统计数据一次获取下来，然后进行数据的统计分析，从而获取结果，获取的粒度是以存储池为基准的。

拿到所有对象的信息：

	for obj in `rados -p rbd ls`;do rados -p rbd stat $obj >> obj.txt;done

这个获取的时间长短是根据对象的多少来的，如果担心出问题，可以换个终端查看进度：
	
	tail -f obj.txt

获取 RBD 的镜像列表：

	rbd -p rbd ls
	img2
	mysql-img
	volume-e5376906-7a95-48bb-a1c6-bb694b4f4813.backup.base

获取 RBD 的镜像的 prefix ：

	root@mon:~# for a in `rbd -p rbd ls`;do echo $a ;rbd -p rbd info $a|grep prefix |awk '{print $2}' ;done
    img2
    rb.0.f4730.2ae8944a
    mysql-img
    rb.0.f4652.2ae8944a
    volume-e5376906-7a95-48bb-a1c6-bb694b4f4813.backup.base
    rbd_data.23a53c28fb938f


获取指定RBD镜像的大小：

	root@mon:~# cat obj.txt |grep rb.0.f4652.2ae8944a |awk  '{ SUM += $6 } END { print SUM/1024/1024 " MB" }'
	52.8047 MB

将上面的汇总，使用脚本一次查询出指定存储池中所有镜像的大小：

	#!/bin/bash
	# USAGE:./get_used.sh <poolname>

	objfile=obj.txt
	Poolname=${1}
	echo "In the pool ${Poolname}":

	for obj in `rados -p $Poolname ls`
	do
    	rados -p $Poolname stat $obj >> $objfile
	done

	for image in `rbd -p $Poolname ls`
	do
    	Imagename=$image
    	Prefix=`rbd  -p $Poolname info $image|grep prefix |awk '{print $2}'`
    	Used=`cat $objfile |grep $Prefix|awk '{ SUM += $6 } END { print SUM/1024/1024 " MB" }'`
    	echo $Imagename $Prefix
    	echo Used: $Used
	done

执行的效果如下：

	root@mon:~# ./get_used.sh rbd
	In the pool rbd:
	img2 rb.0.f4730.2ae8944a
	Used: 3076 MB
	mysql-img rb.0.f4652.2ae8944a
	Used: 158.414 MB
	volume-e5376906-7a95-48bb-a1c6-bb694b4f4813.backup.base rbd_data.23a53c28fb938f
	Used: 96 MB

注意这里只统计了 image 里面的真实容量，如果是用了链接 clone 的，存在容量复用的问题，需要自己看是否需要统计那一部分的对象，方法同上。