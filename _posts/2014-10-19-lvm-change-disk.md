---
layout: post
title: Centos使用LVM扩展根目录容量
category: 技术
tags: [Linux]
keywords: Linux,LVM
description: 
---

>  
安装完集群后，发现有一台DataNode节点在安装系统时没有对分区进行分配，使用系统默认的分配方式，以致HDFS只用到了320G硬盘中的40G空间，本文是通过LVM技术，来对硬盘空间进行动态增加及减少，增加HDFS的空间。

###缩小LVM分区
系统：Centos 6.5 2.6.32-431.el6.x86_64，硬盘使用情况：

```
[root@slave2 ~]# df -lh
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_slave2-lv_root   50G  8.3G   39G  18% /
tmpfs                          1.9G   72K  1.9G   1% /dev/shm
/dev/sda1                      485M   40M  420M   9% /boot
/dev/mapper/vg_slave2-lv_home  241G  188M  228G   1% /home
cm_processes                   1.9G  216K  1.9G   1% /var/run/cloudera-scm-agent/process
```

1、首先卸载所要压缩的分区：/dev/mapper/vg_slave2-lv_home。注：扩容不需要卸载

```
[root@slave2 ~]# umount /dev/mapper/vg_slave2-lv_home
```

2、强制检查/dev/mapper/vg_slave2-lv_home文件系统的正确性

```
[root@slave2 ~]# e2fsck -f /dev/mapper/vg_slave2-lv_home
e2fsck 1.41.12 (17-May-2010)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/vg_slave2-lv_home: 19/15990784 files (0.0% non-contiguous), 1051485/63943680 blocks
```
3、先缩小/dev/mapper/vg_slave2-lv_home文件系统的大小，缩小至10G

```
[root@slave2 ~]# resize2fs /dev/mapper/vg_slave2-lv_home 10G
resize2fs 1.41.12 (17-May-2010)
Resizing the filesystem on /dev/mapper/vg_slave2-lv_home to 2621440 (4k) blocks.
The filesystem on /dev/mapper/vg_slave2-lv_home is now 2621440 blocks long.
```

4、再缩小LVM大小

```
[root@slave2 ~]# lvresize -L 10G /dev/mapper/vg_slave2-lv_home
  WARNING: Reducing active logical volume to 10.00 GiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce lv_home? [y/n]: y
  Reducing logical volume lv_home to 10.00 GiB
  Logical volume lv_home successfully resized
```
5、挂载所有分区：

```
[root@slave2 ~]# mount -a
```

6、检查各个分区的容量：可见挂载/home已经缩小为10G容量

```
[root@slave2 ~]# df -h
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_slave2-lv_root   50G  8.3G   39G  18% /
tmpfs                          1.9G   72K  1.9G   1% /dev/shm
/dev/sda1                      485M   40M  420M   9% /boot
cm_processes                   1.9G  216K  1.9G   1% /var/run/cloudera-scm-agent/process
/dev/mapper/vg_slave2-lv_home  9.9G  165M  9.2G   2% /home
```
###增加LVM分区
通过上面减少LVM分区步骤，已经压缩出200多G未使用的空间，下面动态的扩展根目录容量。

```
[root@slave2 ~]# vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  vg_slave2   1   3   0 wz--n- 297.60g 233.93g
```
1、使用lvexted命令向根目录增加233.92G(注：增加233.93G是报空间不足的总是，暂时减少0.01G来解决)

```
[root@slave2 ~]# lvextend -L +233.92G  /dev/mapper/vg_slave2-lv_root
  Rounding size to boundary between physical extents: 233.92 GiB
  Extending logical volume lv_root to 283.92 GiB
  Logical volume lv_root successfully resized
```
2、使用resize2fs，来改变文件系统容量分配

```
[root@slave2 ~]# resize2fs /dev/mapper/vg_slave2-lv_root
resize2fs 1.41.12 (17-May-2010)
Filesystem at /dev/mapper/vg_slave2-lv_root is mounted on /; on-line resizing required
old desc_blocks = 4, new_desc_blocks = 18
Performing an on-line resize of /dev/mapper/vg_slave2-lv_root to 74428416 (4k) blocks.
The filesystem on /dev/mapper/vg_slave2-lv_root is now 74428416 blocks long.
```
3、查看改变后的磁盘容量，可以看到根目录空间已经增加，大小为280G

```
[root@slave2 ~]# df -h
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_slave2-lv_root  280G  8.3G  258G   4% /
tmpfs                          1.9G   72K  1.9G   1% /dev/shm
/dev/sda1                      485M   40M  420M   9% /boot
cm_processes                   1.9G  216K  1.9G   1% /var/run/cloudera-scm-agent/process
/dev/mapper/vg_slave2-lv_home  9.9G  165M  9.2G   2% /home
```