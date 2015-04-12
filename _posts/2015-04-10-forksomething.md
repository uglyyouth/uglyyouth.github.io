---
layout: post
title: "Linux内核创建一个新进程的过程"
quote: "进程是计算机中已运行程序的实体。在面向线程设计的系统（Linux 2.6及更新的版本）中，进程本身不是基本运行单位，而是线程的容器。"
image: false
video: false
---
> 此文仅用于MOOC`Linux内核分析`作业
> 
> **张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


---- 


# task\_struct数据结构
根据[wiki][1]的定义,**进程**是计算机中已运行程序的实体。在面向线程设计的系统（Linux 2.6及更新的版本）中，进程本身不是基本运行单位，而是线程的容器。

> A computer program is a passive collection of instructions; a **process** is the actual execution of those instructions. Several processes may be associated with the same program; for example, opening up several instances of the same program often means more than one process is being executed.

在Linux中,**task\_struct**其实就是通常所说的PCB。该结构定义位于:

\~\~\~
/include/linux/sched.h
\~\~\~

task\_struct比较庞大,大致可以分为几个部分:

- 进程状态（State）
- 进程调度信息（Scheduling Information）
- 各种标识符（Identifiers）
- 进程通信有关信息（IPC：Inter\_Process Communication）
- 时间和定时器信息（Times and Timers）
- 进程链接信息（Links）
- 文件系统信息（File System）
- 虚拟内存信息（Virtual Memory）
- 页面管理信息（page）
- 对称多处理器（SMP）信息
- 和处理器相关的环境（上下文）信息（Processor Specific Context）
- 其它信息

其中比较重要的几个参数:

- `volatile long state;`进程状态,可见`/include/linux/sched.h`文件中的宏,TASK\_RUNNING等
- `unsigned int rt_priority;`实时优先级
- `unsigned int policy;`调度策略
- `pid_t pid;`进程标识符
- `struct task_struct __rcu *real_parent;`real parent
- `struct list_head children;`list of my children
- `struct files_struct *files;`系统打开文件



# 分析内核处理过程sys\_clone

fork、vfork和clone三个系统调用实际上都是通过`do_fork`来实现进程的创建.
见如下语句:

{% highlight c %}
return do\_fork(clone\_flags, newsp, 0, parent\_tidptr, child\_tidptr);
{% endhighlight %}

- **do\_fork** 函数

而do\_fork函数真正实现复制是`copy_process`

{% highlight c %}
long do\_fork(unsigned long clone\_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{

	...
	
	p = copy_process(clone_flags, stack_start, stack_size,
	         child_tidptr, NULL, trace);
  ...
}
{% endhighlight %}


- **copy\_process**函数

**copy\_process()**主要完成进程数据结构，各种资源的初始化。


{% highlight c %}
p = dup\_task\_struct(current);
{% endhighlight %}

  1. (省略的IF语句)检查clone\_flags参数,防止无效的组合进入
  2. `p = dup_task_struct(current);`调用dup\_task\_struct()为新进程创建一个内核栈
  3. 判断权限及允许范围的代码
  4. 对子进程的描述符初始化和复制父进程的资源给子进程
	- `retval = sched_fork(clone_flags, p);`完成调度相关的设置，将这个task分配给CPU
	- `if (retval)`语句群,复制共享进程的的各个部分
	- `retval = copy_thread(clone_flags, stack_start, stack_size, p);`复制父进程堆栈的内容到子进程的堆栈中去.这其中,copy\_thread()函数中的语句`p->thread.ip = (unsigned long) ret_from_fork;`决定了新进程的**第一条指令地址**.

- **dup\_task\_struct()**

{% highlight c %}
static struct task\_struct *dup\_task\_struct(struct task\_struct *orig)
{
	struct task_struct *tsk;
	struct thread_info *ti;
	int node = tsk_fork_get_node(orig);
	int err;
	
	tsk = alloc_task_struct_node(node);
	if (!tsk)
	    return NULL;
	
	ti = alloc_thread_info_node(tsk, node);
	if (!ti)
	    goto free_tsk;
	
	err = arch_dup_task_struct(tsk, orig);
	if (err)
	    goto free_ti;
	
	tsk->stack = ti;
# ifdef CONFIG\_SECCOMP

	tsk->seccomp.filter = NULL;
# endif

	setup_thread_stack(tsk, orig);
	clear_user_return_notifier(tsk);
	clear_tsk_need_resched(tsk);
	set_task_stack_end_magic(tsk);

# ifdef CONFIG\_CC\_STACKPROTECTOR
	tsk->stack_canary = get_random_int();
# endif

	atomic_set(&tsk->usage, 2);
# ifdef CONFIG\_BLK\_DEV\_IO\_TRACE
	tsk->btrace_seq = 0;
# endif
	tsk->splice_pipe = NULL;
	tsk->task_frag.page = NULL;
	
	account_kernel_stack(ti, 1);
	
	return tsk;

free\_ti:
	free_thread_info(ti);
free\_tsk:
	free_task_struct(tsk);
	return NULL;
}
{% endhighlight %}

  1. `tsk = alloc_task_struct_node(node);`为task\_struct开辟内存
  2. `ti = alloc_thread_info_node(tsk, node);`ti指向thread\_info的首地址，同时也是系统为新进程分配的两个连续页面的首地址。
  3. `err = arch_dup_task_struct(tsk, orig);`复制父进程的task\_struct信息到新的task\_struct里, (*dst = *src;)
  4. `tsk->stack = ti;`task的对应栈
  5. `setup_thread_stack(tsk, orig);`初始化thread info结构
  6. `set_task_stack_end_magic(tsk);`栈结束的地址设置数据为栈结束标示(for overflow detection)


# gdb跟踪sys\_clone

- 用GDB来跟踪sys\_clone,设置以下**断点**:
![fork1][image-1]

- 运行后首先停在sys\_clone处:
![fork2][image-2]

- 然后是do\_fork,之后是copy\_process:
![fork3][image-3]

- 进入copy\_thread:
![fork4][image-4]

 - 在copy\_thread中,我们可以查看**p**的值
 ![fork5][image-5]

- 但是回到copy\_process后再查看,将得到一个`value optimized out`的提示,这是因为Linux内核打开gcc的-O2选项优化导致.如果想要关掉,可以参考:[这里][2]
![fork6][image-6]

- ret\_from\_fork按照之前的分析被调用,跟踪到`syscall_exit`后无法继续.如果想在本机调试system call，那么当你进入system call时，系统已经在挂起状态了。如果想要**跟踪调试**system\_call,可以使用kgdb等
![fork7][image-7]




# 新进程是从哪里开始执行的？
在之前的分析中,谈到copy\_process中的`copy_thread()`函数,正是这个函数决定了子进程从系统调用中**返回**后的执行.

{% highlight c %}
int copy\_thread(unsigned long clone\_flags, unsigned long sp,
	unsigned long arg, struct task_struct *p)
{
	...
	
	*childregs = *current_pt_regs();
	childregs->ax = 0;
	if (sp)
	    childregs->sp = sp;
	
	p->thread.ip = (unsigned long) ret_from_fork;
	
	...
}
{% endhighlight %}


#### 子进程执行ret\_from\_fork

{% highlight asm %}
ENTRY(ret\_from\_fork)
	CFI_STARTPROC
	pushl_cfi %eax
	call schedule_tail
	GET_THREAD_INFO(%ebp)
	popl_cfi %eax
	pushl_cfi $0x0202       # Reset kernel eflags
	popfl_cfi
	jmp syscall_exit
	CFI_ENDPROC
END(ret\_from\_fork)
{% endhighlight %}

执行起点与内核堆栈如何保证一致?

1. 在ret\_from\_fork之前,也就是在copy\_thread()函数中`*childregs = *current_pt_regs();`该句将父进程的regs参数赋值到子进程的内核堆栈,
2. \*childregs的类型为pt\_regs,里面存放了SAVE ALL中压入栈的参数
3. 故在之后的RESTORE ALL中能顺利执行下去.


# 总结
1. Linux通过复制父进程来创建一个新进程,通过调用**do_fork**来实现
2. Linux为每个新创建的进程动态地分配一个`task_struct`结构.
3. 为了把内核中的所有进程组织起来，Linux提供了几种组织方式，其中哈希表和双向**循环链表**方式是针对系统中的所有进程（包括内核线程），而运行队列和等待队列是把处于同一状态的进程组织起来
4. fork()函数被调用一次，但返回**两次**

## 参考

1. [dup\_task\_struct][3]
2. [Linux进程切换以及内核线程的返回值][4]
3. [kernel-book][5]

[1]:	http://en.wikipedia.org/wiki/Process_(computing)
[2]:	http://www.lenky.info/archives/2013/03/2238
[3]:	http://blog.csdn.net/sunnybeike/article/details/6882280
[4]:	http://www.oschina.net/question/234345_47879
[5]:	http://www.kerneltravel.net/kernel-book/%E6%B7%B1%E5%85%A5%E5%88%86%E6%9E%90Linux%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81.html

[image-1]:	/media/2015-4-10/fork1.png
[image-2]:	/media/2015-4-10/fork2.png
[image-3]:	/media/2015-4-10/fork3.png
[image-4]:	/media/2015-4-10/fork4.png
[image-5]:	/media/2015-4-10/fork5.png
[image-6]:	/media/2015-4-10/fork6.png
[image-7]:	/media/2015-4-10/fork7.png