title: docker 基础工作原理（二）
date: 2015-01-15 19:22:30
tags: docker 基础工作原理
category: docker
---
在本篇博文中将主要介绍docker使用device mapper管理镜像的原理。这部分内容我也没完全搞懂，以下内容主要是通过参考多篇博文总结出的概要。
##loop设备介绍

在类 UNIX 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件。
在使用之前，一个 loop 设备必须要和一个文件进行连接。这种结合方式给用户提供了一个替代块特殊文件的接口。因此，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。
<!-- more -->
####loop设备测试实例
1. 首先创建一个 1G 大小的空文件：
```shell
 dd if=/dev/zero of=loopfile.img bs=1G count=1
```

2.对该文件格式化为 ext4 格式：
```shell
    # mkfs.ext4    loopfile.img
    mke2fs 1.41.11 (14-Mar-2010)
    loopfile.img is not a block special device.
    Proceed anyway? (y,n) y
    Filesystem label=
    OS type: Linux
    Block size=4096 (log=2)
    Fragment size=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    65536 inodes, 262144 blocks
    13107 blocks (5.00%) reserved for the super user
    First data block=0
    Maximum filesystem blocks=268435456
    8 block groups
    32768 blocks per group, 32768 fragments per group
    8192 inodes per group
    Superblock backups stored on blocks:
    32768,98304, 163840, 229376
    Writing inode tables:done
    Creating journal (8192 blocks): done
    Writing superblocks and filesystem accounting information:done
    This filesystem will be automatically checked every 38 mountsor
    180 days, whichever comesfirst. Use tune2fs -c or -i tooverride.
```
用 file 命令查看下格式化后的文件类型：
```shell
    # file loopfile.img
    loopfile.img: Linux rev 1.0 ext4 filesystem data,UUID=a9dfb4a0-6653-4407-ae05-7044d92c1159 (extents) (large files)(huge files)
```
准备将上面的文件挂载起来：
```shell
# mkdir /mnt/loopback
# mount -o loop loopfile.img /mnt/loopback
```
mount 命令的 -o loop 选项可以将任意一个 loopback 文件系统挂载。上面的 mount 命令实际等价于下面两条命令：
```shell
# losetup /dev/loop0   loopfile.img
# mount /dev/loop0    /mnt/loopback
```
因此实际上，mount -o loop 在内部已经默认的将文件和 /dev/loop0 挂载起来了。然而对于第一种方法(mount -oloop)并不能适用于所有的场景。比如，我们想创建一个硬盘文件，然后对该文件进行分区，接着挂载其中一个子分区，这时就不能用 -oloop 这种方法了。因此必须如下做：
```shell
# losetup /dev/loop1   loopfile.img
# fdisk /dev/loop1
```
卸载挂载点
```shell
# umount /mnt/loopback
```
##docker存储结构分析
docker中镜像的layer是通过device mapper thin provisioning（自动精简配置）模式实现的。
thin provision是在 kernel3.2 中引入的。它主要有以下一些特点：
（1）允许多个虚拟设备存储在相同的数据卷中，从而达到共享数据，节省空间的目的；
（2）支持任意深度的快照。之前的实现的性能为O(n)，新的实现通过一个单独的数据避免了性能随快照深度的增加而降低。
（3）支持元数据存储到单独的设备上。这样就可以将元数据放到镜像设备或者更快的SSD上。
thin provisioning模式需要两个块设备，一个存储data信息，一个存储metadata信息。设置好这两个块设备之后thin provision可以建立一个存储池（pool）。
具体到docker，docker在/var/lib/docker/devicemapper自动创建两个稀疏文件：data文件（默认100G）和metadata文件（默认2G）。稀疏文件只有在真正往里面存入数据后才会实际占用磁盘空间。这两个文件与2个loop设备进行连接，一般来说data文件关联到/dev/loop0设备，metadata文件关联到/dev/loop1设备。thin provison通过这两个设备建立起一个存储池。可以通过dmsetup table查看存储池信息
```shell
[root@localhost devicemapper]# dmsetup table
docker-253:1-153295518-pool: 0 209715200 thin-pool 7:1 7:0 128 32768 1 skip_block_zeroing 
centos-home: 0 1628528640 linear 8:3 8161280
centos-swap: 0 8159232 linear 8:3 2048
centos-root: 0 104857600 linear 8:3 1636689920
```
pool的具体信息如下：
```
docker-253:1-153295518-pool: 0 209715200 thin-pool 7:1 7:0 128 32768 1
```

可以通过dmsetup命令具体分析其table格式：
```
dmsetup create pool \
	--table "0 209715200 thin-pool $metadata_dev $data_dev \
		 $data_block_size $low_water_mark
```
这个存储池大小为209715200/2/1024/1024=100G，metadata设备为7：1，在我的电脑中对应/dev/loop1，data设备为7：0 对应/dev/loop0设备。128指data_block_size，它代表一次申请的最小磁盘空间，单位为sector（512字节）。32768是low_water_mark，单位也是sector。当data的实际剩余空间小于这个值时将会触发一个dm event。详细说明请参考官方文档：https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt

当启动容器后将会从该存储池中分配出10G(默认值)空间创建thin volume，用来存储镜像和容器文件系统的数据。这个过程具体是通过快照(snapshot)实现的。因此同一个镜像可以通过快照被多个容器使用，当容器修改镜像时，通过copy on write实现镜像的修改。
综合来看，docker的存储结构如下图所示

![docker存储结构](/img/docker-devicemapper.png)

比如创建一个apache容器时devicemapper处理流程如下所示：
1.Create a snapshot of the base device.
2.Mount it and apply the changes in the fedora image.
3.Create a snapshot based on the fedora device.
4.Mount it and apply the changes in the apache image.
5.Create a snapshot based on the apache device.
6.Mount it and use as the root in the new container.
其中base device是所有容器的祖先镜像，在docker中可以具体参考setupBaseImage函数。简单来说base device是一个空的格式化过的设备，其格式只能为xfs或者ext4，大小默认为10G，在docker server端启动时可以通过dm.basesize设定。因为base device是所有的容器的共同祖先镜像，一旦大小设定，在不重新启动docker server时docker无法改变其大小（当然也可以自己手动通过device mapper扩充容量，可以参考[使用 Device Mapper来改变Docker容器的大小](http://www.cnblogs.com/feisky/p/4106004.html)）

##总结
目前只对docker如何使用devicemapper有了大致框架的了解，具体细节还没完全搞懂，后面再根据docker源码详细分析。



###参考文章
[1][loop device介绍及losetup使用](http://wushank.blog.51cto.com/3489095/1212647)
[2]http://www.projectatomic.io/docs/filesystems/
[3]http://www.cnblogs.com/feisky/p/4106212.html
[4]https://hustcat.github.io/docker-devicemapper2/
[5][Adventures in Docker land](http://blogs.gnome.org/alexl/2013/10/15/adventures-in-docker-land/)
