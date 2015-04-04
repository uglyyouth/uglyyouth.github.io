---
layout: post
title: "system_call中断处理过程"
quote: "采用系统调用机制为用户进程提供一个获得操作系统服务的统一接口层，这样一来可简化用户进程的实现，把一些共性的、繁琐的、与硬件相关、与特权指令相关的任务放到操作系统层来实现，但提供一个简洁的接口给用户进程调用；"
image: false
video: false
---

>此文仅用于MOOC`Linux内核分析`作业
>
>**张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


*****



#一.调试sys_mkdir
根据上周使用的C以及嵌入式汇编实现系统调用的代码改写:[menuOS](https://github.com/mengning/menu),对test.c进行改写，增加**mkdir1**(c语言实现)，**mkdir2**(嵌入式汇编实现)。具体代码如下：


{% highlight c %}

#include <sys/stat.h>
#include <sys/types.h>



int mkdir1(int argc, const char * argv[]) {
    // insert code here...
    int intr;
    mode_t mode=S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH;


   intr=mkdir("temp",mode);


    if(intr==-1)
    {
    printf("directory create Fail!intr=%d\n",intr);
    return -1;
    }

    printf("the actual number of bytes %d \n",intr);

  return 0;
}

int mkdir2(int argc, const char * argv[]) {
    // insert code here...
    int intr;
    mode_t mode=S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH;

    asm volatile ("int $0x80"
                  : "=a" (intr)
                  : "a" (39), "b" ("temp-asm"), "c" (mode)
                  : "memory");


    if(intr==-1)
    {
    printf("directory create Fail!intr=%d\n",intr);
    return -1;
    }

    printf("the actual number of bytes %d \n",intr);

  return 0;
}

int main()
{

    MenuConfig("mkdir","create a directory",mkdir1);
    MenuConfig("mkdir-asm","create a directory(asm)",mkdir2);
  
}

{% endhighlight %}

- 执行后，效果如下：
![syscall1](/media/2015-4-4/syscall1.png)

- 使用gdb调试，设置断点`sys_mkdir`
![syscall2](/media/2015-4-4/syscall4.png)

- 如果继续单步运行，接下来的代码无法调试
![syscall3](/media/2015-4-4/syscall5.png)

- 关于System_call的调试
Linux的内核和System Call不好调试。简单来说，如果想在本机调试system call，那么当你进入system call时，系统已经在挂起状态了。如下所说：

>Debugging on Linux is implemented via the ptrace(2) system call; ptrace can only inspect and stop processes running in userspace. I would expect that FreeBSD's process debugging mechanism is similar, and only designed to work on userspace processes: because the OS kernel will acquire and release locks as well as respond to interrupts quickly, designing the kernel to allow full-featured debugging from userspace seems very unlikely.

如果想要**跟踪调试**system_call(kgdb),可以使用：

1. UML方式
2. 使用串口／别的机器

PS:具体参考：[step into system call source code](http://stackoverflow.com/questions/5999205/cannot-step-into-system-call-source-code),这里不再赘述。


#二.分析

`sys_call`实现的汇编代码：

{% highlight c %}

ENTRY(system_call)
	RING0_INT_FRAME			# can't unwind into user space anyway
	ASM_CLAC
	pushl_cfi %eax			# save orig_eax
	SAVE_ALL
	GET_THREAD_INFO(%ebp)
					# system call tracing in operation / emulation
	testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp)
	jnz syscall_trace_entry
	cmpl $(NR_syscalls), %eax
	jae syscall_badsys
syscall_call:
	call *sys_call_table(,%eax,4)
syscall_after_call:
	movl %eax,PT_EAX(%esp)		# store the return value
syscall_exit:
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	testl $_TIF_ALLWORK_MASK, %ecx	# current->work
	jne syscall_exit_work

restore_all:
	TRACE_IRQS_IRET
restore_all_notrace:
#ifdef CONFIG_X86_ESPFIX32
	movl PT_EFLAGS(%esp), %eax	# mix EFLAGS, SS and CS
	# Warning: PT_OLDSS(%esp) contains the wrong/random values if we
	# are returning to the kernel.
	# See comments in process.c:copy_thread() for details.
	movb PT_OLDSS(%esp), %ah
	movb PT_CS(%esp), %al
	andl $(X86_EFLAGS_VM | (SEGMENT_TI_MASK << 8) | SEGMENT_RPL_MASK), %eax
	cmpl $((SEGMENT_LDT << 8) | USER_RPL), %eax
	CFI_REMEMBER_STATE
	je ldt_ss			# returning to user-space with LDT SS
#endif
restore_nocheck:
	RESTORE_REGS 4			# skip orig_eax/error_code
irq_return:
	INTERRUPT_RETURN
.section .fixup,"ax"
ENTRY(iret_exc)
	pushl $0			# no error code
	pushl $do_iret_error
	jmp error_code
.previous
	_ASM_EXTABLE(irq_return,iret_exc)

#ifdef CONFIG_X86_ESPFIX32
	CFI_RESTORE_STATE
ldt_ss:
#ifdef CONFIG_PARAVIRT
	/*
	 * The kernel can't run on a non-flat stack if paravirt mode
	 * is active.  Rather than try to fixup the high bits of
	 * ESP, bypass this code entirely.  This may break DOSemu
	 * and/or Wine support in a paravirt VM, although the option
	 * is still available to implement the setting of the high
	 * 16-bits in the INTERRUPT_RETURN paravirt-op.
	 */
	cmpl $0, pv_info+PARAVIRT_enabled
	jne restore_nocheck
#endif

/*
 * Setup and switch to ESPFIX stack
 *
 * We're returning to userspace with a 16 bit stack. The CPU will not
 * restore the high word of ESP for us on executing iret... This is an
 * "official" bug of all the x86-compatible CPUs, which we can work
 * around to make dosemu and wine happy. We do this by preloading the
 * high word of ESP with the high word of the userspace ESP while
 * compensating for the offset by changing to the ESPFIX segment with
 * a base address that matches for the difference.
 */
#define GDT_ESPFIX_SS PER_CPU_VAR(gdt_page) + (GDT_ENTRY_ESPFIX_SS * 8)
	mov %esp, %edx			/* load kernel esp */
	mov PT_OLDESP(%esp), %eax	/* load userspace esp */
	mov %dx, %ax			/* eax: new kernel esp */
	sub %eax, %edx			/* offset (low word is 0) */
	shr $16, %edx
	mov %dl, GDT_ESPFIX_SS + 4 /* bits 16..23 */
	mov %dh, GDT_ESPFIX_SS + 7 /* bits 24..31 */
	pushl_cfi $__ESPFIX_SS
	pushl_cfi %eax			/* new kernel esp */
	/* Disable interrupts, but do not irqtrace this section: we
	 * will soon execute iret and the tracer was already set to
	 * the irqstate after the iret */
	DISABLE_INTERRUPTS(CLBR_EAX)
	lss (%esp), %esp		/* switch to espfix segment */
	CFI_ADJUST_CFA_OFFSET -8
	jmp restore_nocheck
#endif
	CFI_ENDPROC
ENDPROC(system_call)

{% endhighlight %}


1. 这段代码首先做了保存上下文的工作，主要是通过一个宏`SAVE ALL`实现的(还有pushl_cfi %eax).在中断发生前夕，要把所有相关寄存器的内容都保存在堆栈中,这是通过SAVE_ALL宏完成的
2. `GET_THREAD_INFO(%ebp)` 将当前信息保存在ebp
3. `testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp)  
  jnz syscall_trace_entry `
  判断是否 trace调用
4. `	cmpl $(NR_syscalls), %eax
	jae syscall_badsys`判断系统调用号是否超出最大值
5. `call *sys_call_table(,%eax,4)`系统调用的数字实际上是一个序列号，表示其在系统的一个数组sys_call_table[]中的位置。
7. `movl %eax,PT_EAX(%esp)`保存系统调用的返回值
8. `DISABLE_INTERRUPTS(CLBR_ANY)  ` 屏蔽其他系统调用
9. `movl TI_flags(%ebp), %ecx` 寄存器ecx是通用寄存器,在保护模式中，可以作为内存偏移指针（此时，DS作为 寄存器或段选择器），此时为返回到系统调用之前做准备
10. `testl $_TIF_ALLWORK_MASK, %ecx     jne syscall_exit_work`退出系统调用之前,检查是否需要处理信号  
11. `RESTORE_REGS 4`x86架构恢复寄存器代码
12. `INTERRUPT_RETURN` 即iret。（The following will never clobber any registers:INTERRUPT_RETURN (aka. "iret")）




#总结

1. 使用中断方式的系统调用的简单流程：
![syscall process](https://www.ibm.com/developerworks/linux/library/l-system-calls/figure1.gif)
![inline4](https://www.ibm.com/developerworks/linux/library/l-system-calls/figure2.gif)
2. **系统调用的过程**其实是另一种上下文切换的实现。
  - 首先`SAVE ALL`保存上下文
  - 根据IDT调用内核函数
  - 执行`RESTORE_ALL`并返回用户模式
3. 从系统的调用过程可以看出，与中断处理相似，在进行操作前后，均要对现场进行**保护和恢复**
4. 与用户态的函数库调用执行过程相比，系统调用执行过程的有四点主要的不同：
  - 不是通过“CALL”指令而是通过“INT”指令发起调用；
  - 不是通过“RET”指令，而是通过“IRET”指令完成调用返回；
  - 当到达内核态后，操作系统需要严格检查系统调用传递的参数，确保不破坏整个系统的安全性；
  - 执行系统调用可导致进程等待某事件发生，从而可引起进程切换；

##参考
1. [内核调试](http://my.oschina.net/fgq611/blog/113249)
