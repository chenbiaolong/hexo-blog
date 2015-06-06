title: docker实现原理之namespace
date: 2015-02-26 19:26:25
tags:  docker 基础工作原理
category: docker
---

##概要

传统上，linux很多资源是全局管理的，例如系统中所有的进程是通过pid标识的，这意味着内核管理着一个全局pid表，进程号必须为唯一的。类似的还有内核的文件系统挂载点数据信息、用户ID号等。我们知道，要实现虚拟化必须要有独立的资源分配，才能使容器之间不互相影响，那如何使这些全局表局域化呢？答案是namespace。Namespace将传统的全局资源变为某个名字空间的局域资源。目前linux内核实现的namespace主要有:
<!-- more -->
1.Mount namespace(CLONE_NEWNS):系统挂载点
2.UTS namespace (CLONE_NEWUTS):Hostname等信息
3.IPC namespace(CLONE_NEWIPC):进程间通讯
4.PID namespace(CLONE_NEWPID):进程号
5.Network namespace(CLONE_NEWNET):网络相关资源
6.User namespace(CLONE_NEWUSER):用户ID
可以看出，以上的这些系统资源在没有引入namespace时是由内核全局管理的。linux内核为了支持容器虚拟化功能，加入了以上6种namespace，实现这些全局系统资源局域化，使每一个namespace空间都拥有独立的一套系统资源。由于本文主要讲述docker虚拟化的实现原理，考虑到篇幅，将主要从内核角度简介linux的PID namespace。PID namespace使属于不同的名字空间的进程可以拥有相同的进程号，对实现docker的虚拟化至关重要。

##namespace内核相关结构

在task_struct 结构中有一个结构体指针nsproxy。nsproxy结构体定义了内核支持的namespace。
```c
struct task_struct {
...
/* namespaces */
    struct nsproxy *nsproxy;
...
}
```
nsproxy的定义如下(Linux/include/linux/nsproxy.h)：
```c
struct nsproxy {
         atomic_t count;
         struct uts_namespace *uts_ns;
         struct ipc_namespace *ipc_ns;
         struct mnt_namespace *mnt_ns;
         struct pid_namespace *pid_ns_for_children;
         struct net           *net_ns;
};
```
由于我们选取的内核源码版本为3.19，因此在上面代码中还未实现User namespace，user namespace在内核3.8才被实现。
系统中有一个默认的nsproxy:init_nspoxy，该结构在task初始化是也会被初始化。
```c
#define INIT_TASK(tsk)  \
{
...
         .nsproxy   = &init_nsproxy,
...
}
```
init_nsproxy定义如下：
```c
struct nsproxy init_nsproxy = {
         .count                  = ATOMIC_INIT(1),
         .uts_ns                 = &init_uts_ns,
  #if defined(CONFIG_POSIX_MQUEUE) || defined(CONFIG_SYSVIPC)
          .ipc_ns                 = &init_ipc_ns,
  #endif
          .mnt_ns                 = NULL,
          .pid_ns_for_children    = &init_pid_ns,
  #ifdef CONFIG_NET
          .net_ns                 = &init_net,
  #endif
  };
```

##PID namespace解析

在这些namespace中，我们选取pid_namespace作为本文分析的重点，其他namespace的原理也大致相同。
我们分析一下pid_namespace结构体：
```c
struct pid_namespace {
 struct kref kref;
 struct pidmap pidmap[PIDMAP_ENTRIES];
 int last_pid; //创建新进程时使用，局部于该命名空间的新进程的进程号
 struct task_struct *child_reaper;
 //相当于该命名空间的init进程，负责所有隶属于该命名空间进程死亡时的资源回收
 struct kmem_cache *pid_cachep;
 unsigned int level; //该命名空间所在的层次，最高为0
 struct pid_namespace *parent; //上一级命名空间
#ifdef CONFIG_PROC_FS
 struct vfsmount *proc_mnt;
#endif
#ifdef CONFIG_BSD_PROCESS_ACCT
 struct bsd_acct_struct *bacct;
#endif
};
```
重点关注其中的struct pidmap结构体，struct pidmap用来标志一个进程号是否被使用。在上面的分析中，我们知道当内核启动时所有的进程都被分配到一个初始的namespace中，对于pid namespace，这个初始名字空间就是init_pid_ns,定义如下。
```c
 /*
   * PID-map pages start out as NULL, they get allocated upon
   * first use and are never deallocated. This way a low pid_max
   * value does not cause lots of bitmaps to be allocated, but
   * the scheme scales to up to 4 million PIDs, runtime.
   */
  struct pid_namespace init_pid_ns = {
          .kref = {
                  .refcount       = ATOMIC_INIT(2),
          },
          .pidmap = {
                  [ 0 ... PIDMAP_ENTRIES-1] = { ATOMIC_INIT(BITS_PER_PAGE), NULL }
          },
          .last_pid = 0,
          .nr_hashed = PIDNS_HASH_ADDING,
          .level = 0,
          .child_reaper = &init_task,
          .user_ns = &init_user_ns,
          .ns.inum = PROC_PID_INIT_INO,
  #ifdef CONFIG_PID_NS
          .ns.ops = &pidns_operations,
  #endif
  };
```
init_pid_ns的名字空间层级为0，没有父名字空间。pidmap被初始化为0，表示目前还没有id号被分配出去。last_pid设置为0，表示新分配的pid号将会从进程1开始。另外child_reaper在init_pid_ns被设置为init_task,该进程负责所有隶属于该命名空间进程死亡时的资源回收。init_task这个进程比较特殊，是linux内核启动时的进程1，是所有进程的祖先，在内核启动时它进行了一系列环境初始化操作：比如执行linux的启动脚本等。此外它负责初始名字空间里所有僵尸进程的资源回收。在这里init_pid_ns与子名字空间有一点区别：因为init_pid_ns是最初的名字空间，它是在内核加载完成之初建立的，因此它的child_reaper设定的init_task拥有系统环境初始化的作用；而后续通过clone建立的新名字空间，由于此时内核已经加载完成，因此新的名字空间的进程1是没有环境初始化作用的。后续将对子名字空间的进程1进行一些分析。
新的名字空间的建立是通过clone函数实现的。clone函数最终会调用do_fork系统调用，最终调用create_new_namespace函数。其调用层次如下图所示：
![此处输入图片的描述][1]
在执行copy_process时，我们找到名字空间对该名字空间里的进程1的处理代码段：
```c
/*
* is_child_reaper returns true if the pid is the init process
* of the current namespace. As this one could be checked before
* pid_ns->child_reaper is assigned in copy_process, we check
* with the pid number.
*/

static struct task_struct *copy_process(...) {
...
if (is_child_reaper(pid)) { //当前进程在当前名字空间是进程1
//进程1负责回收当前名字空间的僵尸进程
    ns_of_pid(pid)->child_reaper = p; 
//设置为不可被杀死
    p->signal->flags |= SIGNAL_UNKILLABLE; 
 }
 ...
 }
```
可以看出，对于名字空间的进程1，内核将该进程赋值给该名字空间的child_reaper使之负责该名字空间的僵尸进程资源回收。同时将该进程设置为SIGNAL_UNKILLABLE，标志该进程不可被杀死。需要注意的是不可杀死这个属性只在本名字空间有效，对于其父名字空间来说该进程只是一个普通的进程，仍然可被杀死。与init_pid_ns不同的是，子名字空间的进程1并没有类似于init_task进程的环境初始化操作。因此当新建一个docker容器时，容器里的进程1并不会执行bashrc等初始化脚本。事实上，docker的一些设备初始化、环境变量初始化操作是由docker程序手动完成的，而不是由进程1完成的。
由于加入了pid namespace，因此同一个进程在不同的名字空间有不同的pid号。内核通过alloc_pid函数分配进程在各个名字空间的PID号，具体代码如下：
```c
struct pid *alloc_pid(struct pid_namespace *ns) {
...//省略一些代码
for (i = ns->level; i >= 0; i--) {  
//tmp为当前遍历到的namespace，使用其独立的位图分配pid值  
        nr = alloc_pidmap(tmp);  
        if (nr < 0)  
            goto out_free;  
  
        pid->numbers[i].nr = nr;  
        pid->numbers[i].ns = tmp;  
        tmp = tmp->parent;  
    }    
...//省略一些代码
}
```

##小结

本文主要对pid namespace进行分析，从内核代码实现角度论述namespace的实现机理。总的来说，namespace将linux内核的一些全局资源局域化，它是docker实现虚拟化的核心基础。通过namespace，linux的一些传统全局资源就可以被具体某个名字空间独立占有，各个名字空间的资源不会相互冲突，起到环境隔离的作用。需要注意的是namespace隔离的系统资源是类似于PID号等内核资源，而非CPU 内存等实际物理资源。docker的物理资源限制和控制是通过Cgroup实现的。



  [1]: http://7u2qr4.com1.z0.glb.clouddn.com/blog_%E5%9B%BE%E7%89%877.jpg