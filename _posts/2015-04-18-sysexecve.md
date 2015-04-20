---
layout: post
title: "Linux内核如何装载和启动一个可执行程序"
quote: "ELF在计算机科学中，是一种用于执行档、目的档、共享库和核心转储的标准文件格式。"
image: false
video: false
---
>此文仅用于MOOC`Linux内核分析`作业
>
>**张依依**+原创作品转载请注明出处 + 《Linux内核分析》**MOOC课程**http://mooc.study.163.com/course/USTC-1000029000


*****


#ELF

[可执行和可链接格式](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)   (Executable and Linkable Format，缩写为ELF)，常被称为`ELF`格式，在计算机科学中，是一种用于执行档、目的档、共享库和核心转储的标准文件格式。

1999年，被86open项目选为x86架构上的类Unix操作系统的**二进制文件格式**标准，用来取代COFF。因其可扩展性与灵活性，也可应用在其它处理器、计算机系统架构的操作系统上。

ELF文件由 ELF header和文件数据组成.文件数据包括:

- Program header table, 程序头：描述段信息
- Section header table, Section头：链接与重定位需要的数据
- Data referred to by entries in the program header table or section header table 如:.text .data

![ELF](/media/2015-4-18/Elf-layout--en.svg.png)


#动态&静态
当用户启动一个应用程序时，它们正在调用一个可执行和链接格式映像。
Linux中ELF支持两种类型的库，每一种库都有各自的优缺点:

- **静态库**包含在编译时静态绑定到一个程序的函数。
- **动态库**则不同，它是在加载应用程序时被加载的，而且它与应用程序是在运行时绑定的。

![libraries](/media/2015-4-18/libraries.gif)

而动态链接分为可执行程序**装载时**动态链接和**运行时**动态链接.


#GDB

设置以下断点:

![sysexecve1](/media/2015-4-18/sysexecve1.png)

在MenuOS执行exec后,中断情况如下:

![sysexecve2](/media/2015-4-18/sysexecve2.png)

![sysexecve3](/media/2015-4-18/sysexecve3.png)

![sysexecve4](/media/2015-4-18/sysexecve4.png)

进入**search_binary_handler**后可以查看一些变量情况:比如fmt,bprm

![sysexecve5](/media/2015-4-18/sysexecve5.png)

![sysexecve6](/media/2015-4-18/sysexecve6.png)

进入**start_thread**后使用po命令,可以看到new_ip处:

![sysexecve9](/media/2015-4-18/sysexecve9.png)

在改变regs前后查看regs:

![sysexecve7](/media/2015-4-18/sysexecve7.png)

![sysexecve8](/media/2015-4-18/sysexecve8.png)

继续跟踪可以看到在执行do_notify_resume后,进入0x08048d0a处,即之前的new_ip处(hello的入口地址):

![sysexecve10](/media/2015-4-18/sysexecve10.png)

![sysexecve11](/media/2015-4-18/sysexecve11.png)


#分析sys_execve

当`sys_execve`被调用后,涉及的主要函数为:

~~~
do_execve -> do_execve_common ->  exec_binprm
~~~

- **syscall**
    {% highlight c %}
        SYSCALL_DEFINE3(execve,
        		const char __user *, filename,
        		const char __user *const __user *, argv,
        		const char __user *const __user *, envp)
        {   //真正执行程序的功能exec.c文件中的do_execve函数中实现
        	return do_execve(getname(filename), argv, envp);

        }

    {% endhighlight %}

- **do_execve**

    {% highlight c %}
        int do_execve(struct filename *filename,
        	const char __user *const __user *__argv,
        	const char __user *const __user *__envp)
        {
        	struct user_arg_ptr argv = { .ptr.native = __argv };
        	struct user_arg_ptr envp = { .ptr.native = __envp };
            //调用do_execve_common
        	return do_execve_common(filename, argv, envp);
        }
    {% endhighlight %}

- **do_execve_common**

    {% highlight c %}

        static int do_execve_common(struct filename *filename,
                        struct user_arg_ptr argv,
                        struct user_arg_ptr envp)
        {
            struct linux_binprm *bprm;
            struct file *file;
            struct files_struct *displaced;
            int retval;

            ..

            //打开要执行的文件，并检查其有效性
            file = do_open_exec(filename);
            retval = PTR_ERR(file);
            if (IS_ERR(file))
                goto out_unmark;

            sched_exec();
            // 填充linux_binprm结构
            bprm->file = file;
            bprm->filename = bprm->interp = filename->name;

            ...

            //将文件名、环境变量和命令行参数拷贝到新分配的页面中
            retval = copy_strings_kernel(1, &bprm->filename, bprm);
            if (retval < 0)
                goto out;

            bprm->exec = bprm->p;
            retval = copy_strings(bprm->envc, envp, bprm);
            if (retval < 0)
                goto out;

            retval = copy_strings(bprm->argc, argv, bprm);
            if (retval < 0)
                goto out;
            //调用exec_binprm,保存当前的pid并且调用 search_binary_handler
            retval = exec_binprm(bprm);
            if (retval < 0)
                goto out;

            /* execve succeeded */
            current->fs->in_exec = 0;
            current->in_execve = 0;
            acct_update_integrals(current);
            task_numa_free(current);
            free_bprm(bprm);
            putname(filename);
            if (displaced)
                put_files_struct(displaced);
            return retval;

        }
            {% endhighlight %}

        
- 关于**linux_binprm**保存要执行的文件相关的参数,包括argc,envc,*filename,*interp等
- `exec_binprm`在保存了bprm后调用该函数来进一步操作,这个函数除了保存pid以外,还执行了`ret = search_binary_handler(bprm);`来查询能够处理相应可执行文件格式的处理器，并调用相应的load_binary方法以启动进程。


- **search_binary_handler**
    {% highlight c %}
            int search_binary_handler(struct linux_binprm *bprm)
        {
            ...
            //循环binary formats handler,直到找到
         retry:
            read_lock(&binfmt_lock);
            list_for_each_entry(fmt, &formats, lh) {
                if (!try_module_get(fmt->module))
                    continue;
                read_unlock(&binfmt_lock);
                bprm->recursion_depth++;
                //解析elf格式执行的位置
                retval = fmt->load_binary(bprm);
                read_lock(&binfmt_lock);
                
            ...
        }
    {% endhighlight %}

- 这里的`fmt`是linux_binfmt格式,该结构用来load the binary formats
- 经由search_binary_handler函数呼叫load_elf_binary函数
- ELF格式的二进制映像的认领、装入和启动是由load_elf_binary()完成的。在`/fs/binfmt_elf.c`中,定义了如下结构:

    {% highlight c %}
            static struct linux_binfmt elf_format = {
            .module     = THIS_MODULE,
            .load_binary    = load_elf_binary,
            .load_shlib = load_elf_library,
            .core_dump  = elf_core_dump,
            .min_coredump   = ELF_EXEC_PAGESIZE,
        };
    {% endhighlight %}


- **load_elf_binary**

    {% highlight c %}
    static int load_elf_binary(struct linux_binprm *bprm)
{
    ...
    
    //获取头
    loc->elf_ex = *((struct elfhdr *)bprm->buf);



    //读取头信息
        if (loc->elf_ex.e_phentsize != sizeof(struct elf_phdr))
        goto out;
    if (loc->elf_ex.e_phnum < 1 ||
        loc->elf_ex.e_phnum > 65536U / sizeof(struct elf_phdr))
        goto out;
    size = loc->elf_ex.e_phnum * sizeof(struct elf_phdr);
    retval = -ENOMEM;
    elf_phdata = kmalloc(size, GFP_KERNEL);
    if (!elf_phdata)
        goto out;
        ...

    //读取可执行文件的解析器
    for (i = 0; i < loc->elf_ex.e_phnum; i++) {
        if (elf_ppnt->p_type == PT_INTERP) {
            ...
    }

    ...

    //如果需要装入解释器，并且解释器的映像是ELF格式的，就通过load_elf_interp()装入其映像，并把将来进入用户空间时的入口地址设置成load_elf_interp()的返回值，那显然是解释器的程序入口。而若不装入解释器，那么这个地址就是目标映像本身的程序入口。
    if (elf_interpreter) {
        unsigned long interp_map_addr = 0;

        elf_entry = load_elf_interp(&loc->interp_elf_ex,
                        interpreter,
                        &interp_map_addr,
                        load_bias);
        if (!IS_ERR((void *)elf_entry)) {
            
            interp_load_addr = elf_entry;
            elf_entry += loc->interp_elf_ex.e_entry;
        }
        if (BAD_ADDR(elf_entry)) {
            retval = IS_ERR((void *)elf_entry) ?
                    (int)elf_entry : -EINVAL;
            goto out_free_dentry;
        }
        reloc_func_desc = interp_load_addr;

        allow_write_access(interpreter);
        fput(interpreter);
        kfree(elf_interpreter);
    } else {
        elf_entry = loc->elf_ex.e_entry;
        if (BAD_ADDR(elf_entry)) {
            retval = -EINVAL;
            goto out_free_dentry;
        }
    }
    {% endhighlight %}


当**load_elf_binary()**执行完毕，返回至do_execve()在返回至sys_execve()时，系统调用的返回地址已经被改写成了被装载的ELF程序的入口地址了。

- 可执行文件开始执行的起点在哪里？

当sys_execve()系统调用从内核态返回到用户态时，EIP寄存器直接跳转到ELF程序的入口地址。

#总结
1. linux通过sys_execve()系统调用从文件系统中读取、识别并加载**elf**
2. 调用sys_execve后,执行过程:do_execve -> do_execve_common -> exec_binprm->load_elf_binary()->sys_close
3. 根据elf的库类型,elf_entry不一样.**load_elf_binary**通过解析器将不同的入口地址写入.

#参考
1. [Linkers and Loaders](http://www.iecc.com/linker/)
2. [ELF在Linux下的加载过程](http://blog.csdn.net/joker0910/article/details/7686836)
