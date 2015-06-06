title: docker 基础工作原理（一）
date: 2015-01-09 15:30:41
tags: docker 基础工作原理
category: docker
---
## 概述

相信很多人和我一样，初学docker时一直无法搞懂docker镜像的工作机理。这几天对docker如何工作进行了一番研究，简单整理一下。
docker的两大核心基础技术是[namespace](http://lwn.net/Articles/531114/)和cgroup，cgroup主要作资源的限制隔离，它可以限制一组进程中能使用的最大资源使用量，相对比较好理解；namespace同样可以实现资源隔离，不同的是它是通过使PID,IPC,Network等系统资源不再是全局性的，而是属于特定的Namespace实现的。每个Namespace里面的资源对其他Namespace都是透明的，这个概念有点类似于linux的多用户机制。
<!-- more -->
namespace的详细介绍可以参考[Introduction to Linux namespaces](https://blog.jtlebi.fr/2013/12/22/introduction-to-linux-namespaces-part-1-uts/)系列文章，国内已经有人对这系列博文进行了翻译：[linux namespace简介](http://blog.lucode.net/linux/intro-Linux-namespace-1.html)，本文接下来对namepace的介绍主要参考以上两篇博文。
现在linux内核中支持的namespace主要有：

>* Mount namespaces (CLONE_NEWNS)
isolate the set of filesystem mount points seen by a group of processes. Thus, processes in different mount namespaces can have different views of the filesystem hierarchy. With the addition of mount namespaces, the mount() and umount() system calls ceased operating on a global set of mount points visible to all processes on the system and instead performed operations that affected just the mount namespace associated with the calling process. 
>* UTS namespaces(CLONE_NEWUTS)
isolate two system identifiers—nodename and domainname—returned by the uname() system call; the names are set using the sethostname() and setdomainname() system calls. In the context of containers, the UTS namespaces feature allows each container to have its own hostname and NIS domain name. This can be useful for initialization and configuration scripts that tailor their actions based on these names. The term "UTS" derives from the name of the structure passed to the uname() system call: struct utsname. The name of that structure in turn derives from "UNIX Time-sharing System"
>* IPC namespaces (CLONE_NEWIPC) 
isolate certain interprocess communication (IPC) resources, namely, System V IPC objects and (since Linux 2.6.30) POSIX message queues. The common characteristic of these IPC mechanisms is that IPC objects are identified by mechanisms other than filesystem pathnames. Each IPC namespace has its own set of System V IPC identifiers and its own POSIX message queue filesystem. 
>* PID namespaces (CLONE_NEWPID, Linux 2.6.24) 
isolate the process ID number space. In other words, processes in different PID namespaces can have the same PID. One of the main benefits of PID namespaces is that containers can be migrated between hosts while keeping the same process IDs for the processes inside the container. PID namespaces also allow each container to have its own init (PID 1), the "ancestor of all processes" that manages various system initialization tasks and reaps orphaned child processes when they terminate. 
>*  Network namespaces (CLONE_NEWNET, started in Linux 2.4.19 2.6.24 and largely completed by about Linux 2.6.29) 
provide isolation of the system resources associated with networking. Thus, each network namespace has its own network devices, IP addresses, IP routing tables, /proc/net directory, port numbers, and so on.Network namespaces make containers useful from a networking perspective: each container can have its own (virtual) network device and its own applications that bind to the per-namespace port number space; suitable routing rules in the host system can direct network packets to the network device associated with a specific container. Thus, for example, it is possible to have multiple containerized web servers on the same host system, with each server bound to port 80 in its (per-container) network namespace. 
>* User namespaces (CLONE_NEWUSER, started in Linux 2.6.23 and completed in Linux 3.8) 
isolate the user and group ID number spaces. In other words, a process's user and group IDs can be different inside and outside a user namespace. The most interesting case here is that a process can have a normal unprivileged user ID outside a user namespace while at the same time having a user ID of 0 inside the namespace. This means that the process has full root privileges for operations inside the user namespace, but is unprivileged for operations outside the namespace. 

在这篇博文中，将主要介绍PID namespace和mount namepace。通过这两个namespace模拟docker在基础文件系统中运行的原理。我们将利用busybox建立一个可以满足linux系统运行的基础环境，并且通过chroot切换根目录，实现环境隔离。

## 利用chroot和busybox实现文件系统隔离

chroot可以实现根路径的切换，但如果新的根路径环境下没有基础库和程序（比如bash），那么chroot将不能正常切换根路径。busybox提供了能保证linux系统正常运行的基础工具（如bash、ls等命令工具），我们可以利用chroot+busybox在我们本地系统中建立一个新的沙箱系统（当然此时并没有名字空间的隔离，只是根文件系统实现了隔离）。
首先从官方[下载busybox的源码](http://www.busybox.net/downloads/)，这里我使用的是1.22版本。解压后直接使用默认配置,然后运行make,make install。
```shell
[root@localhost busybox-1.22.1]# make defconfig #使用默认配置
[root@localhost busybox-1.22.1]# make
[root@localhost busybox-1.22.1]# make install

```
一切顺利的话，系统会在当前路径下生成一个_install文件夹。
```shell
[root@localhost busybox-1.22.1] cd _install
[root@localhost _install]#ls
bin  linuxrc  sbin  usr
```
_install 文件夹中包含了许多基础工具
```shell
[root@localhost _install]# cd bin
[root@localhost bin]# ls
ash      chgrp   cttyhack       dumpkmap  fgrep   hostname  kill     lsattr    more     netstat        printenv   rmdir         setserial  sync    usleep
base64   chmod   date           echo      fsync   hush      linux32  lzop      mount       nice           ps         rpm           sh         tar     vi
busybox  chown   dd             ed        getopt  ionice    linux64  makemime  mountpoint  pidof          pwd        run-parts     sleep      touch   watch
cat      conspy  df             egrep     grep    iostat    ln       mkdir     mpstat      ping           reformime  scriptreplay  stat       true    zcat
catv     cp      dmesg          false     gunzip  ipcalc    login    mknod     mt          ping6          rev        sed           stty       umount
chattr   cpio    dnsdomainname  fdflush   gzip    kbd_mode  ls       mktemp    mv          pipe_progress  rm         setarch       su         uname
```
返回_install路径，尝试进行chroot
```shell
[root@localhost bin]# cd ..
[root@localhost _install]# chroot . /bin/ash
chroot: failed to run command ‘/bin/ash’: No such file or directory
[root@localhost _install]# 
```
这里提示没有找到/bin/ash，但实际上bin文件夹的ash是存在的。这里主要原因是切换根路径后，系统无法找到busybox依赖的动态链接库,busybox无法正常工作。先看一下busybox依赖的库
```shell
[root@localhost _install]# ldd /bin/busybox 
	linux-vdso.so.1 =>  (0x00007fff415fa000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f53f4926000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f53f4565000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f53f4c3d000)
[root@localhost _install]# mkdir lib64
[root@localhost _install]# cp /lib64/libm.so.6 ./lib64/
[root@localhost _install]# cp /lib64/libc.so.6 ./lib64/
[root@localhost _install]# cp /lib64/ld-linux-x86-64.so.2 ./lib64
[root@localhost _install]# chroot . /bin/ash
/ # ls
bin      lib64    linuxrc  sbin     usr
/ # 
```
从11行起可以看出chroot已经成功完成根路径切换，现在运行在busybox搭建的根路径环境下。
但实际上现在根路径并不是完整的，可以看出busybox搭建的根文件系统并没有/proc和/dev等目录，在该环境下允许top等指令并不能正常允许。接下来是namespace发挥作用的时候了。

## namespace隔离

以下内容主要来源于[linux namespace简介](http://blog.lucode.net/linux/intro-Linux-namespace-1.html)系列文章。这里我们直接使用文中的测试代码。
```c
//namespace.c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
// sync primitive
int checkpoint[2];
static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};
int child_main(void* arg) {
  char c;
  // init sync primitive
  close(checkpoint[1]);
  // setup hostname
  printf(" - [%5d] World !\n", getpid());
  sethostname("In Namespace", 12);
  // wait...
  read(checkpoint[0], &c, 1);
  execv(child_args[0], child_args);
  printf("Ooops\n");
  return 1;
}
int main() {
  // init sync primitive
  pipe(checkpoint);
  printf(" - [%5d] Hello ?\n", getpid());
  int child_pid = clone(child_main, child_stack+STACK_SIZE,
      CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
  // further init here (nothing yet)
  // signal "done"
  close(checkpoint[1]);
  waitpid(child_pid, NULL, 0);
  return 0;
}
```
编译namespace.c，建立一个proc的空文件夹。
```shell
[root@localhost _install]# mkdir proc
[root@localhost _install]# gcc namespace.c -o ns
[root@localhost _install]# 
```
执行ns进入新的名字空间，chroot将根目录切换到当前目录下。
```shell
[root@localhost _install]# ./ns
 - [14649] Hello ?
 - [    1] World !
[root@In Namespace _install]# chroot . /bin/ash
/ # 
/ # ps #这个ps没有获得任何进程数据，因为当前/proc为空文件夹
PID   USER     TIME   COMMAND
/ # mount -t proc proc /proc #将当前名字空间的proc文件系统挂在到/proc
/ # ps #获得当前名字空间的进程数据
PID   USER     TIME   COMMAND
    1 0          0:00 /bin/bash
   30 0          0:00 /bin/ash
   33 0          0:00 ps
/ # 
```
现在在当前根路径下已经基本实现了名字空间的隔离和文件系统的隔离，一个沙箱系统的雏形已经实现。docker的运行的基础原理也大致如此，不过现在还没有添加进cgroup。

## busybox 基础镜像制作

现在考虑将当前的busybox环境打成一个基础镜像。
```shell
[root@localhost _install]# tar --numeric-owner -cvf  ../BusyBox-test.tar ./
```
注意，此时的BusyBox-test.tar文件只是一个tar包，里面缺乏docker需要的一些json文件和目录，并不能直接用docker load倒入。
```shell
[root@localhost busybox-1.22.1]# docker load < BusyBox-test.tar
2015/01/09 13:01:51 Error: open /var/lib/docker/tmp/docker-import-114653123/repo/bin/json: no such file or directory
[root@localhost busybox-1.22.1]# 
```
可以使用以下指令导入镜像
```shell
[root@localhost busybox-1.22.1]# cat BusyBox-test.tar | docker import - busyboxtest
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a
[root@localhost busybox-1.22.1]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
busyboxtest         latest              3341f4c8d832        19 minutes ago      4.241 MB
centos              7                   8efe422e6104        3 days ago          224 MB
centos              centos7             8efe422e6104        3 days ago          224 MB
centos              latest              8efe422e6104        3 days ago          224 MB
busybox             buildroot-2014.02   4986bf8c1536        8 days ago          2.433 MB
busybox             latest              4986bf8c1536        8 days ago          2.433 MB
```
运行docker，查看镜像是否能正常运行。
```shell
[root@localhost busybox-1.22.1]# docker run -it  busyboxtest /bin/sh
/ # ls
bin  dev etc lib64  linuxrc namespace.c  ns  proc   sbin    sys  usr
/ # 
```
可以看出镜像能正常运行。将busyboxtest镜像导出，看镜像文件和简单的tar文件有何不同。
```shell
[root@localhost _install]# docker save busyboxtest > busyboxtest-img.tar
[root@localhost _install]# ls
bin  lib64  linuxrc  busyboxtest-img.tar  namespace.c  ns  proc  sbin  usr
```
```shell
[root@localhost _install]# mkdir img
[root@localhost _install]# cp busyboxtest-img.tar img
[root@localhost _install]# cd img
[root@localhost img]# ls
busyboxtest-img.tar
[root@localhost img]# tar xvf busyboxtest-img.tar 
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/VERSION
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/json
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/layer.tar
repositories
```
在镜像文件夹中，layer实际上就是我们打包的tar文件。docker import进tar文件后对tar文件进行了一些处理，方便docker对镜像文件进行管理。
接下来我们对busyboxtest进行修改，往根目录里添加一个文本文件，并将修改后的镜像命名为busyboxtest:v1，解压镜像文件后看文件夹里的内容
```shell
[root@localhost img2]# tar xvf busyboxtestv1.tar 
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/VERSION
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/json
3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a/layer.tar
8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653/
8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653/VERSION
8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653/json
8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653/layer.tar
repositories
[root@localhost img2]# tree
.
|-- 3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a
|   |-- json
|   |-- layer.tar
|   `-- VERSION
|-- 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653
|   |-- json
|   |-- layer.tar
|   `-- VERSION
|-- busyboxtestv1.tar
`-- repositories
[root@localhost img2]# tree
.
|-- 3341f4c8d8326866a15099dd09cff11d3f09c07d50c4d0dec9930b26dfa11b2a
|   |-- json
|   |-- layer.tar
|   `-- VERSION
|-- 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653
|   |-- json
|   |-- layer.tar
|   `-- VERSION
|-- busyboxtestv1.tar
`-- repositories

2 directories, 8 files
[root@localhost img2]# 

```
3341f...这个文件夹保存的是未修改的镜像信息，8d24...文件保存的是修改信息，它只保存和基础镜像的diff信息，具体存储在该文件夹的layer.tar文件。解压8d24...文件夹的layer.tar文件
```shell
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# ls
json  layer.tar  VERSION
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# tar xvf layer.tar 
.ash_history
img_test_file
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# ls
img_test_file  json  layer.tar  VERSION
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# 
```
可以看出里面多了一个img_test_file文件，这个正是我们在基础镜像里添加的文件。.ash_history保存的是命令执行历史记录:
```shell
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# cat .ash_history 
ls
vi img_test_file
ls
exit
[root@localhost 8d24158476ecbdea530dc210bda9b60970dd809d10a914ae19c83e8359396653]# 
```
docker可以根据这个diff信息构建出新的工作环境。

## 总结

docker实现资源隔离的两大linux基础技术是:namespace和cgroup。并且通过chroot实现运行的根目录环境切换。在这篇博文中主要简单模拟了docker的namespace和chroot工作流程，并且介绍了制作基础镜像的过程。本篇博文只是很概要的模拟docker工作原理，具体的详细流程需要继续分析docker的源码。
