---
layout: post
title: "Linux内核启动分析"
quote:
image: false
video: false
---
>此文仅用于MOOC`Linux内核分析`作业

>**张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


*****


#以下为正文

总体来说，Linux的启动过程是复杂的。`CPU`启动时从内存的最高地址获取第一道指令的硬件设计，
即ROM芯片上的**BIOS**。在X86中，即`CS:EIP=FFFF:0000H`。


之后的启动主要分为两个阶段
: 1. bootloader启动阶段
: 2. linux 内核初始化和启动阶段


linux内核启动的阶段主要是从`start_kernel`开始；然后`user_mode`开始，
start_kernel结束;最后加载linux内核完毕，转入`cpu_idle`进程。

本文主要分析内核启动的**第一个阶段**，即从start_kernel到init进程启动；

内核的初始化过程由`start_kernel`函数开始，至第一个用户进程init结束，
调用了一系列的初始化函数对所有的内核组件进行初始化。
其中，start_kernel、rest_init、kernel_init等函数构成了整个初始化过程的主线。

##start_kernel

使用一个[小OS](https://github.com/mengning/menu)来分析内核启动，如下图。

![init1](/media/2015-3-22/init1.png)


使用gdb跟踪调试内核，在start_kernel处设置断点：

![init3](/media/2015-3-22/init3.png)



start_kernel位于内核目录下：`/init/main.c`中

```c

asmlinkage __visible void __init start_kernel(void)
{
	char *command_line;
	char *after_dashes;

	/*
	 * Need to run as early as possible, to initialize the
	 * lockdep hash:
	 */
	lockdep_init();
	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	debug_objects_early_init();

	.....

	ftrace_init();

	/* Do the rest non-__init'ed, we're now alive */
	rest_init();
}

```

在`start_kernel()`中 Linux 将完成整个系统的内核初始化。
内核初始化的最后一步就是启动 init 进程这个所有进程的祖先。


从以上代码中可以看出start_kernel中调用到大量的init函数，来完成内核的各种初始化（省略的部分）。

在函数第一句`lockdep_init();`初始化一张lockdep hash用来实现互斥信号量后，
**第二句**`set_task_stack_end_magic(&init_task);`
语句则把系统中第一个进程（0号进程）作防溢出工作。

这里的`init_task`在Linux中属于一个比较特殊的进程，
它是内核开发者人为制造出来的，而不是其他进程通过do_fork来完成。在`/init/init_task.c`中：


```c

/* Initial task structure */
struct task_struct init_task = INIT_TASK(init_task);
EXPORT_SYMBOL(init_task);

```

这里调用宏`INIT_TASK`完成对`init_task`de赋值，不再赘述。
这里的init_task就是0号进程，通过调试也可以看出：

![init5](/media/2015-3-22/init5.png)


`start_kernel()`在最后会调用**rest_init()**，在这里，我们的1号进程将被创建和运行.


##rest_init

```c

static noinline void __init_refok rest_init(void)
{
	int pid;

	rcu_scheduler_starting();
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.
	 */
	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
}

```



`rest_init`同样位于目录`/init/main.c`中。

从rest_init开始，Linux开始产生进程，因为init_task是静态制造出来的，pid=0，
它试图将从最早的汇编代码一直到start_kernel的执行都纳入到init_task进程上下文中。
在rest_init函数中，内核将通过下面的代码产生第一个真正的进程(pid=1):

`kernel_thread(kernel_init, NULL, CLONE_FS);`





















start_kernel()在最后会调用rest_init()，这个函数会启动一个内核线程来运行kernel_init()，自己则调用cpu_idle()进入空闲循环，让调度器接管控制权。抢占式的调度器就可以周期性地接管控制权，从而提供多任务处理能力。
   kernel_init()用于完成初始化rootfs、加载内核模块、挂载真正的根文件系统。根据Documentation/early-userspace/README的描述，目前2.6的kernel支持三方式来挂载最终的根文件系统：
