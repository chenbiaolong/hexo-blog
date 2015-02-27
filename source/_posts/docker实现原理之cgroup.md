title: docker实现原理之cgroup
date: 2015-02-27 15:11:54
tags:  docker 基础工作原理
category: docker
---
##概要
cgroups是control groups的缩写，是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：cpu,memory,IO等等）的机制。上篇博客写的namespace作用是使linux的全局资源局域化，使各个名字空间的系统环境相互隔离，互不影响。而cgroup可以限制资源使用的最大值，限制当前进程组对外的最大影响（但无法隔离其他进程对自己的影响）。namespace与cgroup相互配合将使容器具备环境隔离和资源限制的能力，再加上镜像提供的根目录环境，使用chroot即可以为容器提供一个隔离的根目录环境。
本文将主要从内核源码的角度分析docker常用的cgroup资源限制功能：cpuset memory 和blkio。
<!-- more -->
##cgroup 子系统介绍
blkio -- 这个子系统为块设备设定输入/输出限制，比如物理设备（磁盘，固态硬盘，USB 等等）。
cpu -- 这个子系统控制cgroup中所有进程可以使用的时间片。
cpuacct -- 这个子系统自动生成 cgroup 中任务所使用的 CPU 报告。
cpuset -- 这个子系统为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点。
devices -- 这个子系统可允许或者拒绝 cgroup 中的任务访问设备。
freezer -- 这个子系统挂起或者恢复 cgroup 中的任务。
memory -- 这个子系统设定 cgroup 中任务使用的内存限制，并自动生成由那些任务使用的内存资源报告。
net_cls -- 这个子系统使用等级识别符（classid）标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包。
ns -- 名称空间子系统。
cgroup系统是一个树状结构，子cgroup的限定条件必须为父cgroup的子集。如下图：
![此处输入图片的描述][1]
管理功能通过 VFS (虚拟文件系统)接口暴露，新建cgroup只需建立文件夹。比如新建一个cpuset管理组，只需要在cpuset目录下新建文件夹
```
mkdir mytest
```
在该目录下会自动生成管理cpuset的文件接口。如下图：
![此处输入图片的描述][2]
直接对文件夹里的文件进行读写操作即可设置相应的资源限制。
```
echo 1 >  mytest/cpuset.cpus
echo 0 >  mytest/cpuset.mems
echo '22751' > mytest/tasks 
```
这三条指令的意思是设定该进程组只能使用cpu1，内存节点使用0，将进程22751加入该cgroup。
接下来我们将以cpuset、mem、blkio为例，简单说明内核是如何进行资源限制的。
##cpuset子系统
我们先看一下cpuset数据结构：
```c
struct cpuset {
        struct cgroup_subsys_state css;
 ...//忽略一些代码
        cpumask_var_t cpus_allowed;
        nodemask_t mems_allowed;
...//忽略一些代码
      
};
```
忽略了一些代码，重点关注cpuset的cpus_allowed。这个结构用于限定一个进程组所能使用cpu核。通过task_struct，进程都能找到进程所在进程组的cpu限制和内存节点限制，具体的内核数据结构比较复杂，这里略过。
linux内核是通过将进程加入对应cpu核的任务队列实现进程与cpu核绑定功能的。新建的进程与已存在的进程调度略有不同。我们知道，内核是通过do_fork实现新建进程，在do_fork函数里调用 wake_up_new_task实现新建进程的调度。wake_up_new_task相关代码如下：
```c
/*
 * wake_up_new_task - wake up a newly created task for the first time.
 *
  * This function will do some initial scheduler statistics housekeeping
  * that must be done for every newly created context, then puts the task
  * on the runqueue and wakes it.
  */
 void wake_up_new_task(struct task_struct *p)
 {
        unsigned long flags;
        struct rq *rq;
        raw_spin_lock_irqsave(&p->pi_lock, flags);
 #ifdef CONFIG_SMP
        /*
         * Fork balancing, do it here and not earlier because:
         *  - cpus_allowed can change in the fork path
         *  - any previously selected cpu might disappear through hotplug
        */
         set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
 #endif
 
         /* Initialize new task's runnable average */
         init_task_runnable_average(p);
         rq = __task_rq_lock(p);
         activate_task(rq, p, 0);
         p->on_rq = TASK_ON_RQ_QUEUED;
         trace_sched_wakeup_new(p, true);
         check_preempt_curr(rq, p, WF_FORK);
 #ifdef CONFIG_SMP
        if (p->sched_class->task_woken)
            p->sched_class->task_woken(rq, p);
 #endif
         task_rq_unlock(rq, p, &flags);
 }
```
从代码中可以看出，内核是通过调用 
```
set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
```
实现进程在特定cpu核上运行的。`select_task_rq`返回的是进程被允许运行的一个合适的cpu id。`set_task_cpu`会使该进程运行在该cpu核上。

对于已经存在的进程，实现的原理也大致相同。对于完全公平调度CFS算法，内核是通过调用`select_task_rq_fair`从允许的cpu核中选择一个合适的cpu id返回，然后加入任务队列，等待调度运行。

##memory子系统
memory子系统是通过linux的resource counter机制实现的。在具体实现的过程中，cgroup通过内核中的resource counter机制实现内存的限制。resource counter相当于一个通用的资源计数器，在内核中通过res_counter结构来描述。该结构可用于记录某类资源的当前使用量、最大使用量以及上限等信息。mem_cgroup定义如下：
```
struct mem_cgroup {
...
// the counter to account for memory usage
 struct res_counter res;
 //the counter to account for mem+swap usage.
struct res_counter memsw;
//the counter to account for kernel memory usage.
struct res_counter kmem;
...
}
```
res_counter定义如下：
```
struct res_counter {
          /*
           * the current resource consumption level
           */
          unsigned long long usage;
          /*
           * the maximal value of the usage from the counter creation
           */
          unsigned long long max_usage;
          /*
           * the limit that usage cannot exceed
           */
          unsigned long long limit;
          /*
           * the limit that usage can be exceed
           */
          unsigned long long soft_limit;
          /*
           * the number of unsuccessful attempts to consume the resource
           */
          unsigned long long failcnt;
          /*
           * the lock to protect all of the above.
           * the routines below consider this to be IRQ-safe
           */
          spinlock_t lock;
          /*
           * Parent counter, used for hierarchial resource accounting
           */
          struct res_counter *parent;
  };
```
res_counter中的每个字段表示对内存使用量的记录。用户态下memory子系统所导出的配置文件与该结构中的字段互相对应，比如mem.limit_in_bytes表示当前cgroup可使用内存的最大上线，该文件与res_counter结构中的limit字段对应。也就是说，当用户在用户态向mem.limit_in_bytes文件写入值后，则res_counter中的limit字段相应更新。

内核对res_counter进行操作时有三个基本函数：res_counter_init()对res_counter进行初始化；当分配资源时，res_counter_charge()记录资源的使用量，并且该函数还会检查使用量是否超过了上限，并且记录当前资源使用量的最大值；当资源被释放时，res_counter_uncharge()则减少该资源的使用量。

`res_counter_charge`调用`res_counter_charge_locked`函数判断当前进程组的资源使用量是否超出限制。
```c
int res_counter_charge_locked(struct res_counter *counter, unsigned long val)
  {
          if (counter->usage + val > counter->limit) {
                  counter->failcnt++;
                  return -ENOMEM;
          }
  
          counter->usage += val;
          if (counter->usage > counter->max_usage)
                  counter->max_usage = counter->usage;
          return 0;
  }
```
从代码中可以看出，当进程组的资源使用量超出limit值时内核将会返回ENOMEN错误，并根据目前的资源使用量决定是否更新max_usage值。
这里需要注意的是memory系统是限制实际分配的物理内存大小,而非应用程序申请的虚拟内存。只有真正申请物理内存时（page demand 或者copy on write等)，该内存才会被计入res_counter。应用程序调用malloc申请的是虚拟内存，实际使用了多少物理内存必须从内核获得：/proc目录。因此，在docker里可以malloc到远大于限制值的虚拟内存，但当实际要使用超过设置值的内存时该进程会被内核的OOM（out of memory ）killer机制杀死，关于OOM机制可以查阅相关的文档，在做相关测试时需要注意虚拟内存大小与实际物理内存大小的区别。

##blkio子系统
blkio子系统可以设置块设备的两个维度：
1.Weight值。可以对不同的设备设置不同的权重，让权重更高的设备执行更多的IO操作
2.IO操作速度上限值。可以设置块设备的最大读写速度。
这里主要分析IO操作的速度限制原理。
cgroup限制IO速度大小的具体原理是设置IO队列，每隔一段时间（throtl_slice ，目前为0.1s）检查是否超过设定值，如果超过就将IO操作加入等待队列，等待一段时间再进行IO操作。
主要控制函数为`tg_may_dispatch`：
```c
 /*
* Returns whether one can dispatch a bio or not. Also returns approx number
  * of jiffies to wait before this bio is with-in IO rate and can be dispatched
 */
 static bool tg_may_dispatch(struct throtl_grp *tg, struct bio *bio,
                             unsigned long *wait)
 {
        bool rw = bio_data_dir(bio);
        unsigned long bps_wait = 0, iops_wait = 0, max_wait = 0;
        /*
          * Currently whole state machine of group depends on first bio
          * queued in the group bio list. So one should not be calling
          * this function with a different bio if there are other bios
          * queued.
          */
         BUG_ON(tg->service_queue.nr_queued[rw] &&
                bio != throtl_peek_queued(&tg->service_queue.queued[rw]));
         /* If tg->bps = -1, then BW is unlimited */
         if (tg->bps[rw] == -1 && tg->iops[rw] == -1) {
                 if (wait)
                         *wait = 0;
                 return true;
         }
         /*
          * If previous slice expired, start a new one otherwise renew/extend
          * existing slice to make sure it is at least throtl_slice interval
          * long since now.
          */
         if (throtl_slice_used(tg, rw))
                 throtl_start_new_slice(tg, rw);
         else {
                 if (time_before(tg->slice_end[rw], jiffies + throtl_slice))
                         throtl_extend_slice(tg, rw, jiffies + throtl_slice);
         }
         if (tg_with_in_bps_limit(tg, bio, &bps_wait) &&
            tg_with_in_iops_limit(tg, bio, &iops_wait)) {
                 if (wait)
                         *wait = 0;
                return 1;
         }
         max_wait = max(bps_wait, iops_wait);
         if (wait)
                *wait = max_wait;
 
         if (time_before(tg->slice_end[rw], jiffies + max_wait))
                throtl_extend_slice(tg, rw, jiffies + max_wait);
 
         return 0;
 }
```
代码注释写的比较清晰。该函数主要判断是否应该下发IO操作，并且获得需要等待的时间。因此blkio子系统限制IO速度是通过限定其平均值实现的。可以推测，当在`throtl_slice`(0.1s)时间内进行远大于设定值的IO操作时，cgroup是无法限制住的，有一定的局限性。事实上我们在IO速度限制测试中也发现了这个问题，如下图：
![此处输入图片的描述][3]
磁盘设备限制读写速度为1MB/s，但当在短时间内有大量的IO操作时磁盘的读写速度在瞬间远超过限定值。随后速度马上变小。

##小结
本文主要分析cgroup的cpuset 子系统 memory子系统和blkio子系统。这三个系统是docker比较常用的资源限定项，docker建立的每一个容器都是一个cgroup控制组，在docker里运行的进程都处于cgroup系统的控制中。docker通过对linux内核cgroup进行封装，达到资源控制的作用。

  [1]: http://7u2qr4.com1.z0.glb.clouddn.com/blog_cgroup1.png
  [2]: http://7u2qr4.com1.z0.glb.clouddn.com/blog_cgroup2.png
  [3]: http://7u2qr4.com1.z0.glb.clouddn.com/blog_TimLine%E6%88%AA%E5%9B%BE20150227145303.png