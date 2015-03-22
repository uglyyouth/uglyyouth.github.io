---
layout: post
title: "Linux内核启动分析"
quote:
image: false
video: false
---
>此文仅用于MOOC`Linux内核分析`作业
>
>**张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


*****


#以下为正文

总体来说，Linux的启动过程是复杂的。`CPU`启动时从内存的最高地址获取第一道指令的硬件设计，
即ROM芯片上的**BIOS**。在X86中，即`CS:EIP=FFFF:0000H`。


之后的启动主要分为两个阶段：
: a.bootloader启动阶段
: b.linux 内核初始化和启动阶段


linux内核启动的阶段主要是从`start_kernel`开始；然后`user_mode`开始，
start_kernel结束;最后加载linux内核完毕，转入`cpu_idle`进程。


#一.start_kernel


本文主要分析内核启动的**第一个阶段**，即从**start_kernel到init进程**启动；



###调试运行
- 使用一个小的[menuOS](https://github.com/mengning/menu)来分析内核启动，如下图:


![init1](/media/2015-3-22/init1.png)


- 使用gdb跟踪调试内核，在start_kernel处设置断点：


![init3](/media/2015-3-22/init3.png)


###分析
- start_kernel位于内核目录下：`/init/main.c`中

~~~ c

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

~~~~

在`start_kernel()`中 Linux 将完成整个系统的内核初始化。
内核初始化的最后一步就是启动 init 进程这个所有进程的祖先。


从以上代码中可以看出start_kernel中调用到**大量的**init函数，来完成内核的各种初始化（省略的部分）。

其中：

- 第一句`lockdep_init();`初始化一张lockdep hash用来实现互斥信号量后，
- **第二句**`set_task_stack_end_magic(&init_task);`
则把系统中第一个进程（0号进程）作防溢出工作。
  - 这里的`init_task`在Linux中属于一个**比较特殊**的进程，
它是内核开发者人为制造出来的，而不是其他进程通过do_fork来完成。在`/init/init_task.c`中：


~~~ c

/* Initial task structure */
struct task_struct init_task = INIT_TASK(init_task);
EXPORT_SYMBOL(init_task);

~~~

###关于`init_task`

1. 可以看出调用宏`INIT_TASK`完成对`init_task`的赋值，不再赘述。
2. 这里的init_task就是0号进程，通过调试也可以看出：(init_task.pid=0)

![init5](/media/2015-3-22/init5.png)


**`start_kernel()`在最后会调用rest_init()，在这里，我们的1号进程将被创建和运行.**



*******


#二.rest_init

从rest_init开始，Linux开始产生进程，因为init_task是静态制造出来的，pid=0，
它试图将从最早的汇编代码一直到start_kernel的执行都纳入到init_task进程上下文中。
在rest_init函数中，内核将通过下面的代码产生第一个真正的进程(pid=1):


`rest_init`同样位于目录`/init/main.c`中。

~~~ c

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

~~~


###关于`kernel_thread(kernel_init, NULL, CLONE_FS);`
1. 创建一个内核线程,实际上就是内核进程, Linux内核是不支持类似 WindowsNT一样的线程概念的。
Linux本质上只支持进程。
2. `Kernel_thread`调用了do_fork来创建一个进程。这里的kernel_init函数:

~~~ c

static int __ref kernel_init(void *unused)
{
	int ret;

  ...


	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}


  ...


~~~

这里的`run_init_process`就是通过 execve()来运行kernel_init 程序。

这里首先运行“/sbin/init”,如果失败再运行“/etc/init”,然后是 “/bin/init”,然后是“/bin/sh”(也就是说,init 可执行文件可以放在上面代码中寻找的 4 个目录中都可以),如果都失败,则可以通过在系统 启动时在添加的启动参数来指定 init,比如 init=/home/wzhou/init。这里是内核初始化结束并开始用户态初始化的阴阳界。

###关于1号进程

调试rest_init函数，单步跟踪到`pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);`语句，临时变量pid记录了进程号：

![init6](/media/2015-3-22/init6.png)

可以看出，这个时候1号进程已经被创建了。
如果再接着执行到`kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);`可以看到这个时候pid已经变为2了。
（kthreadd的用途是管理和调度其他内核线程）

![init7](/media/2015-3-22/init7.png)


###关于`cpu_idle_loop`

rest_init最后`cpu_startup_entry(CPUHP_ONLINE);`：
~~~ c

void cpu_startup_entry(enum cpuhp_state state)
{
  ...
	arch_cpu_idle_prepare();
	cpu_idle_loop();
}

~~~

此函数是一个一个while(1)循环，即为我们的0号进程。

在循环中它将会调用schedule函数以便在运行队列中有新进程加入时切换到该新进程上。



*******


#总结


1. 正如前面所分析的那样，内核的初始化过程由`start_kernel`函数开始，
至第一个用户进程init结束，
调用了一系列的初始化函数对所有的内核组件进行初始化。
其中，**start_kernel、rest_init、kernel_init**等函数构成了整个初始化过程的主线。
2. start_kernel()在最后会调用rest_init()，
这个函数会启动一个内核线程来运行kernel_init()，
自己则调用cpu_idle()进入空闲循环，让调度器接管控制权。
抢占式的调度器就可以周期性地接管控制权，从而提供多任务处理能力。
3. kernel_init()用于完成初始化rootfs、加载内核模块、挂载真正的根文件系统。
