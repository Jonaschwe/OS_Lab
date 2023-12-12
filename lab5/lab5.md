#### 练习1: 加载应用程序并执行
你需要补充load_icode的第6步，建立相应的用户内存空间来放置应用程序的代码段、数据段等,需设置正确的trapframe内容。

![屏幕截图 2023-12-11 201928.png](https://s2.loli.net/2023/12/11/yhvdOoPMLCj1NJ7.png)

tf->gpr.sp = USTACKTOP;：将线程的栈指针（Stack Pointer）设置为USTACKTOP。USTACKTOP是用户栈的顶部地址，表示该线程在用户态执行时使用的栈空间的顶部。通过将栈指针设置为USTACKTOP，可以确保线程在用户态执行时使用正确的栈空间。tf->epc = elf->e_entry;：将线程的程序计数器（Program Counter）设置为ELF文件的入口地址。ELF文件是可执行文件的一种格式，包含了程序的指令和数据。e_entry字段表示ELF文件的入口地址，即程序执行的第一条指令的地址。通过将程序计数器设置为ELF文件的入口地址，可以确保线程在启动时从正确的位置开始执行。tf->status = (sstatus & ~SSTATUS_SPP) | SSTATUS_SPIE;：设置线程的状态寄存器（Status Register）。具体操作如下：使用位运算符&和~将sstatus寄存器中的SPP位清除，保留其他位的值。SPP位表示当前特权级，清除该位可以将线程切换到用户态。使用位运算符|将清除SPP位后的值与SSTATUS_SPIE进行逻辑或操作，将SPIE位设置为1。SPIE位表示线程在中断处理时是否启用中断，将其设置为1表示允许中断。将上述结果存储到线程的状态寄存器（status字段）中，完成对线程状态的初始化。


请简要描述这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

* 调用schedule函数，调度器占用了CPU的资源之后，用户态进程调用了exec系统调用，从而转入到了系统调用的处理例程；

* 之后进行正常的中断处理例程，然后控制权转移到了syscall.c中的syscall函数，然后根据系统调用号转移给了sys_exec函数，在该函数中调用了do_execve函数来完成指定应用程序的加载；

* 在do_execve中进行了若干设置，包括推出当前进程的页表，换用内核的PDT，调用load_icode函数完成对整个用户线程内存空间的初始化，包括堆栈的设置以及将ELF可执行文件的加载，之后通过current->tf指针修改了当前系统调用的trapframe，使得最终中断返回的时候能够切换到用户态，并且同时可以正确地将控制权转移到应用程序的入口处；

* 在完成了do_exec函数之后，进行正常的中断返回的流程，由于中断处理例程的栈上面的eip已经被修改成了应用程序的入口处，而CS上的CPL是用户态，因此iret进行中断返回的时候会将堆栈切换到用户的栈，并且完成特权级的切换，并且跳转到要求的应用程序的入口处；

* 开始执行应用程序的第一条代码。

#### 练习2：父进程复制自己的内存空间给子进程（需要编码）

创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。

请在实验报告中简要说明你的设计实现过程。

`kernel_thread`函数实现了进程的创建，其中调用了`do_fork`函数，`do_fork`又调用了`copy_mm`实现父进程到子进程的复制，其中又调用`dup_mmap`实现内存管理结构体的复制，它通过循环调用`copy_range`函数实现这一功能。

`copy_range`增加代码如下，通过获取page对应的虚拟地址实现将页表的内容复制到新页表中，然后去建立虚拟地址到物理地址的映射。
```cpp
void * kva_src = page2kva(page);	//获取老页表的值
void * kva_dst = page2kva(npage);	//获取新页表的值
memcpy(kva_dst, kva_src, PGSIZE);	//复制操作
ret = page_insert(to, npage, start, perm);
```

如何设计实现Copy on Write机制？给出概要设计，鼓励给出详细设计。

本实验中`copy_mm`只是完成了进程的复制，可以通过父进程的共享实现COW机制，即子进程并非是父进程复制产生，而是通过指向父进程的指针实现资源共享，并且将其设置为只读。在读操作时可以通过指针实现资源访问，在写操作时会触发异常，在异常处理的过程中执行深拷贝，为子进程单独创建一份资源。

#### 练习3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。并回答如下问题：

* 请分析fork/exec/wait/exit的执行流程。重点关注哪些操作是在用户态完成，哪些是在内核态完成？内核态与用户态程序是如何交错执行的？内核态执行结果是如何返回给用户程序的？
* 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

```lua
+------------------+          +------------------+
|   PROC_UNINIT    |          |    PROC_ZOMBIE   |
|                  |          |                  |
+------------------+          +------------------+
        |                              ^
        v                              |
+------------------+          +------------------+
|  PROC_RUNNABLE   | <------> |      RUNNING     |
|                  |          |                  |
+------------------+          +------------------+
        ^                              ^
        |                              |
        v                              v
+------------------+          +------------------+
| PROC_SLEEPING    | <------> |                  |
|                  |          |                  |
+------------------+          +------------------+


```

一个进程的生命周期如下：

| 状态切换                     | 产生切换的事件                                                         | 切换方式                                                            |
| ---------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------- |
| PROC_UNINIT                  | 创建进程                                                               | do_fork()                                                           |
| PROC_UNINIT->PROC_RUNNABLE   | 创建进程自动切换                                                       | 内核进程，在proc_init()<br />用户进程，在do_fork()调用wakeup_proc() |
| PROC_RUNNABLE->RUNNING       | 当前用户进程退出RUNNING状态<br />内核进行init_main()和cpu_idle()的执行 | 调用schedule()                                                      |
| RUNNING->PROC_RUNNABLE       | yield()系统调用<br />时间片用完                                        | do_yield()                                                          |
| PROC_RUNNABLE->PROC_SLEEPING | wait()系统调用                                                         | do_wait()                                                           |
| PROC_SLEEPING->PROC_RUNNABLE | I/O系统调用<br />进程event completion被唤醒                            |                                                                     |
| RUNNING->PROC_SLEEPING       | I/O系统调用<br />进程进入wait event                                    |                                                                     |
| PROC_SLEEPING->RUNNING       | 子进程唤醒父进程                                                       |                                                                     |
| RUNNING->PROC_ZOMBIE         | exit()系统调用<br />kill()系统调用                                     | do_exit()<br />do_kill()                                            |

分析四个主要函数，大体过程如下：

1. 用户态发起一个系统调用sys_xxxx()
2. sys_xxxx()调用syscall(int64_tnum, ...)
3. syscall首先对传入的参数进行解析，接下来asm volatile进行内联汇编，内联汇编里调用ecall指令
4. ecall接收a[]参数并陷入到内核态，进行中断处理，具体的就是调用syscall()函数。这个syscall是kern/syscall文件夹下的，与第二点的syscall（位于user/libs文件夹）不是一个函数
   ```
   case CAUSE_USER_ECALL:
               //cprintf("Environment call from U-mode\n");
               tf->epc += 4;
               syscall();
               break;
   ```
5. syscall取出参数a[]，通过函数指针syscalls来找到对应的调用sys_xxxx()
6. sys_xxxx()找到所需的参数后，调用do_xxxx()执行具体的处理流程。在后文中进行分析。
7. 执行完成后，从do_xxxx函数开始反向将结果传递，trap.c中断处理完成后进入到epc下一条指令执行，同时返回值通过用户态syscall函数中内联汇编定义的返回值保存地址ret最终传递回用户程序。

**1.do_fork()函数**

练习二与lab4均对该函数进行过分析

```c
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
  
    //分配并初始化线程控制块
    proc=alloc_proc();
    if(proc==NULL){
        goto fork_out;  
    }
    proc->parent=current;
    assert(current->wait_state==0); 
    //分配并初始化内核栈
    if(setup_kstack(proc)!=0){
        goto bad_fork_cleanup_proc;
    }
    //根据clone_flags决定是复制还是共享内存管理系统
    if(copy_mm(clone_flags,proc)!=0){
        goto bad_fork_cleanup_kstack;
    }
    //设置进程的中断帧和上下文
    copy_thread(proc, stack, tf);
    //把设置好的进程加入链表
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid=get_pid();
        hash_proc(proc);  
        set_links(proc);
    }
    local_intr_restore(intr_flag);
    //将新建的进程设为就绪态
    wakeup_proc(proc);
    //将返回值设为线程id
    ret=proc->pid;



fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

主要进行以下过程：

1. 检查是否可以进行fork，也就是当前进程数量是否超过了最大限制
2. 初始化进程，通过lab1-4的函数实现，分配并初始化控制块，内核栈，内存管理系统，中断帧和上下文
3. 加入进程管理的链表，并变为就绪态
4. 在成功创建时，返回进程id；在发生错误时，进行相应的清理工作（释放内核栈和进程控制块），并返回错误码。


**2.do_exec()函数**

练习一对该函数进行过分析。该函数目前没有在提供给用户的lib中出现。

do_exec()函数负责回收进程自身所占用的空间，之后调用load_icode函数，用新的程序覆盖内存空间，形成一个执行新程序的新进程，同时设置好中断帧，使得中断返回后能够跳转到新的进程的入口处执行。

把新的程序加载到当前进程里的工作都在 load_icode() 函数里完成，load_icode()函数按照elf文件结构加载文件中的进程信息。

do_exec()与load_icode()里面只是构建了用户程序运行的上下文，但是并没有完成切换。上下文切换实际上要借助中断处理的返回来完成。当用户态采用exec()（未实现）时，ecall的返回会自动切换好进程。

关于该函数的其他问题：

系统cpu_idle()--->proc_init()--->init_main()--->user_main()的顺序，执行了内核进程user_main()，而内核进程user_main通过宏定义逐层调用do_exec()函数加载了一个exit用户程序。

```c
#define __KERNEL_EXECVE(name, binary, size) ({                          \
            cprintf("kernel_execve: pid = %d, name = \"%s\".\n",        \
                    current->pid, name);                                \
            kernel_execve(name, binary, (size_t)(size));                \
        })

#define KERNEL_EXECVE(x) ({                                             \
            extern unsigned char _binary_obj___user_##x##_out_start[],  \
                _binary_obj___user_##x##_out_size[];                    \
            __KERNEL_EXECVE(#x, _binary_obj___user_##x##_out_start,     \
                            _binary_obj___user_##x##_out_size);         \
        })

#define __KERNEL_EXECVE2(x, xstart, xsize) ({                           \
            extern unsigned char xstart[], xsize[];                     \
            __KERNEL_EXECVE(#x, xstart, (size_t)xsize);                 \
        })

#define KERNEL_EXECVE2(x, xstart, xsize)        __KERNEL_EXECVE2(x, xstart, xsize)

// user_main - kernel thread used to exec a user program
static int
user_main(void *arg) {
#ifdef TEST
    KERNEL_EXECVE2(TEST, TESTSTART, TESTSIZE);
#else
    KERNEL_EXECVE(exit);
#endif
    panic("user_main execve failed.\n");
}
```

但是对于内核态使用do_exec系列函数时，kernel_execve内联汇编不能使用ecall产生中断，于是通过ebreak命令手动开启中断，并设置寄存器的值为10，来在内核态模拟用户态的syscall发起“系统调用”，进而调用do_exec等函数。

```
case CAUSE_BREAKPOINT:
            cprintf("Breakpoint\n");
            if(tf->gpr.a7 == 10){
                tf->epc += 4; //注意返回时要执行ebreak的下一条指令
                syscall();
            }
            break;
```


**3.do_wait()函数**

wait函数主要负责在子进程退出时让父进程完成对进程所占剩余资源的彻底回收。

代码如下：

```c
do_wait(int pid, int *code_store) {
    // 获取当前进程的内存管理结构
    struct mm_struct *mm = current->mm;

    // 如果指定了存储退出码的地址，并且地址无效，则返回错误
    if (code_store != NULL) {
        if (!user_mem_check(mm, (uintptr_t)code_store, sizeof(int), 1)) {
            return -E_INVAL;
        }
    }

    // 定义进程结构体指针和临时变量
    struct proc_struct *proc;
    bool intr_flag, haskid;

repeat:
    haskid = 0;

    // 如果指定了等待的子进程ID（pid不为0）
    if (pid != 0) {
        // 查找指定pid的进程
        proc = find_proc(pid);

        // 如果找到了进程，并且它的父进程是当前进程
        if (proc != NULL && proc->parent == current) {
            haskid = 1;

            // 如果找到的进程是僵尸态，直接跳到 found 标签处
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }
    // 如果等待任意子进程（pid为0）
    else {
        // 遍历当前进程的子进程链表
        proc = current->cptr;

        for (; proc != NULL; proc = proc->optr) {
            haskid = 1;

            // 如果找到的进程是僵尸态，直接跳到 found 标签处
            if (proc->state == PROC_ZOMBIE) {
                goto found;
            }
        }
    }

    // 如果有子进程但没有找到符合条件的子进程，则当前进程进入睡眠状态
    if (haskid) {
        current->state = PROC_SLEEPING;
        current->wait_state = WT_CHILD;
        schedule();

        // 如果当前进程正在退出，则调用 do_exit 函数
        if (current->flags & PF_EXITING) {
            do_exit(-E_KILLED);
        }
        goto repeat;
    }

    // 如果没有子进程，则返回错误
    return -E_BAD_PROC;

found:
    // 如果找到符合条件的子进程
    if (proc == idleproc || proc == initproc) {
        panic("wait idleproc or initproc.\n");
    }

    // 如果指定了存储退出码的地址，将退出码写入该地址
    if (code_store != NULL) {
        *code_store = proc->exit_code;
    }

    // 进入临界区
    local_intr_save(intr_flag);
    {
        // 从进程哈希表和链表中移除该进程
        unhash_proc(proc);
        remove_links(proc);
    }
    // 离开临界区
    local_intr_restore(intr_flag);

    // 释放进程使用的内核栈
    put_kstack(proc);

    // 释放进程结构体
    kfree(proc);

    // 返回成功
    return 0;
}

```

因此，过程如下：

1. 检查参数的有效性
2. 如果指定了等待的子进程ID（pid不为0），且找到了该进程，且处于僵尸态，则进行清理工作
3. 如果等待任意子进程（pid为0），遍历出一个僵尸态的子进程清理
4. 如果有子进程，但以上两步均没有找到符合条件的子进程，也就是子进程仍在运行，则当前进程从running进入wait状态（WT_CHILD），直到子进程进入僵尸状态再被唤醒，goto repeat重新执行2-4处理子进程
5. 等待子进程的过程中，如果当前进程本身需要退出则执行相应的退出操作；如果当前进程被唤醒后发现子进程的状态依然不是僵尸状态，就需要重新进入等待状态，直到等到子进程的退出。
6. 清理子进程工作，从进程哈希表和链表中移除该进程，释放内核栈与结构体

**4.do_exit()函数**

用于在进程自身退出时释放所占用的大部分内存空间，包括内存管理信息如页表等所占空间，同时唤醒父进程完成对自身不能回收的剩余空间的回收，并切换到其他进程。

代码如下：

```c
int do_exit(int error_code) {
    // 检查当前进程是否是空闲进程，如果是则触发 panic（内核恐慌）
    if (current == idleproc) {
        panic("idleproc exit.\n");
    }

    // 检查当前进程是否是初始化进程，如果是则触发 panic
    if (current == initproc) {
        panic("initproc exit.\n");
    }

    // 获取当前进程的内存管理结构
    struct mm_struct *mm = current->mm;

    // 如果内存管理结构不为空
    if (mm != NULL) {
        // 切换回内核的页目录
        lcr3(boot_cr3);

        // 如果内存管理结构的引用计数减为0
        if (mm_count_dec(mm) == 0) {
            // 退出进程的内存映射
            exit_mmap(mm);

            // 释放页目录
            put_pgdir(mm);

            // 销毁内存管理结构
            mm_destroy(mm);
        }

        // 将当前进程的内存管理结构置为NULL
        current->mm = NULL;
    }

    // 将当前进程的状态置为僵尸态
    current->state = PROC_ZOMBIE;

    // 设置当前进程的退出码
    current->exit_code = error_code;

    // 进入临界区
    bool intr_flag;
    struct proc_struct *proc;
    local_intr_save(intr_flag);
    {
        // 获取当前进程的父进程
        proc = current->parent;

        // 如果父进程处于等待子进程状态
        if (proc->wait_state == WT_CHILD) {
            // 唤醒父进程
            wakeup_proc(proc);
        }

        // 遍历当前进程的子进程链表
        while (current->cptr != NULL) {
            proc = current->cptr;
            current->cptr = proc->optr;

            // 重新链接子进程到初始化进程的子进程链表
            proc->yptr = NULL;
            if ((proc->optr = initproc->cptr) != NULL) {
                initproc->cptr->yptr = proc;
            }
            proc->parent = initproc;
            initproc->cptr = proc;

            // 如果子进程是僵尸态，唤醒初始化进程
            if (proc->state == PROC_ZOMBIE) {
                if (initproc->wait_state == WT_CHILD) {
                    wakeup_proc(initproc);
                }
            }
        }
    }
    // 离开临界区
    local_intr_restore(intr_flag);

    // 调度新的进程运行
    schedule();

    // 永远不应该执行到这里
    panic("do_exit will not return!! %d.\n", current->pid);
}

```

过程如下：

1. 检查参数合法性，也就是当前进程是否是两个内核进程
2. 如果内存管理结构不为空，通过内存管理的函数，释放mmap，pgdir，mm，并进入僵尸态
3. 处于僵尸态后，如果父进程也在等待自己进入僵尸态，唤醒父进程处理剩余资源。否则等待之后父进程的do_wait处理剩余资源
4. 处理自己的子进程（孤儿进程），重新链接子进程到init进程的子进程链表。如果子进程是僵尸态，唤醒初始化进程处理子进程剩余资源。
5. 由于当前进程由running退出，调用schedule唤醒一个其它进程。

#### 扩展练习 Challenge
2. 说明该用户程序是何时被预先加载到内存中的？与我们常用操作系统的加载有何区别，原因是什么？

ucore中，用户程序是通过execv系统调用时被预先加载到内存中的，而常用的操作系统是在程序启动时被加载到内存中。原因在于预先加载用户程序可以简化用户程序的启动过程，加快执行速度。