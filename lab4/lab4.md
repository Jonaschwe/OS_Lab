## 练习一：分配并初始化一个进程控制块

#### proc_struct
![屏幕截图 2023-11-17 000544.png](https://s2.loli.net/2023/11/17/mB4AgzR2TOdK1Nc.png)


* enum proc_state state：枚举类型，表示进程的状态，比如运行、就绪、阻塞等。
* int pid：进程的ID，用来唯一标识一个进程。
* int runs：记录进程的运行次数。
* uintptr_t kstack：进程的内核栈，用于处理内核态的函数调用和中断处理。
* volatile bool need_resched：一个布尔值，表示是否需要重新调度来释放CPU。
* struct proc_struct *parent：指向父进程的指针。
* struct mm_struct *mm：指向进程地址空间管理结构的指针，用于管理进程的虚拟内存空间。
* struct context context：保存进程的上下文信息，用于进行进程切换。
* struct trapframe *tf：指向当前中断的 trap frame 结构体的指针。
* uintptr_t cr3：CR3 寄存器的数值，表示页目录表的基地址。
* uint32_t flags：用于存储进程的一些标志信息。
* char name[PROC_NAME_LEN + 1]：存储进程名称的字符数组。
* list_entry_t list_link：用于将进程链接到进程列表中的链表节点。
* list_entry_t hash_link：用于将进程链接到哈希表中的链表节点。


#### alloc_proc函数初始化过程
![屏幕截图 2023-11-17 225301.png](https://s2.loli.net/2023/11/17/P8zxrkMsopm9qLJ.png)

#### struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？

* struct context：该结构包含了处理器寄存器的值，用于在进程之间进行上下文切换。当内核需要切换到一个新的进程时，它会将当前进程的寄存器状态保存在当前进程的PCB中，并将下一个进程的寄存器状态加载到处理器寄存器中。这样，处理器就可以从当前进程切换到下一个进程，而不会丢失任何重要的数据。
* struct trapframe：该结构包含了中断处理程序所需的所有信息，包括中断类型、错误代码、中断发生时的处理器寄存器状态等。当处理器接收到一个中断时，它会将当前进程的寄存器状态保存在当前进程的PCB中，并将中断处理程序所需的寄存器状态加载到处理器寄存器中。这样，处理器就可以执行中断处理程序，处理完后再回到当前进程。

## 练习二：为新创建的内核线程分配资源
proc_init是线程的创建与初始化函数，它初始化了idleproc和initproc。
#### proc_init：
```cpp
void
proc_init(void) {
    int i;

    // 初始化全局的线程控制块双向链表
    list_init(&proc_list);
    // 初始化全局的线程控制块hash表
    for (i = 0; i < HASH_LIST_SIZE; i ++) {
        list_init(hash_list + i);
    }

    // 分配idle线程结构
    if ((idleproc = alloc_proc()) == NULL) {
        panic("cannot alloc idleproc.\n");
    }

    // check the proc structure
    int *context_mem = (int*) kmalloc(sizeof(struct context));
    memset(context_mem, 0, sizeof(struct context));
    int context_init_flag = memcmp(&(idleproc->context), context_mem, sizeof(struct context));

    int *proc_name_mem = (int*) kmalloc(PROC_NAME_LEN);
    memset(proc_name_mem, 0, PROC_NAME_LEN);
    int proc_name_flag = memcmp(&(idleproc->name), proc_name_mem, PROC_NAME_LEN);

    if(idleproc->cr3 == boot_cr3 && idleproc->tf == NULL && !context_init_flag
        && idleproc->state == PROC_UNINIT && idleproc->pid == -1 && idleproc->runs == 0
        && idleproc->kstack == 0 && idleproc->need_resched == 0 && idleproc->parent == NULL
        && idleproc->mm == NULL && idleproc->flags == 0 && !proc_name_flag
    ){
        cprintf("alloc_proc() correct!\n");
    }

    // 为idle线程进行初始化
    idleproc->pid = 0; // idle线程pid作为第一个内核线程，其不会被销毁，pid为0
    idleproc->state = PROC_RUNNABLE; // idle线程被初始化时是就绪状态的
    idleproc->kstack = (uintptr_t)bootstack; // idle线程是第一个线程，其内核栈指向bootstack
    idleproc->need_resched = 1; // idle线程被初始化后，需要马上被调度
    // 设置idle线程的名称
    set_proc_name(idleproc, "idle");
    nr_process ++;

    // current当前执行线程指向idleproc
    current = idleproc;

    // 初始化第二个内核线程initproc， 用于执行init_main函数，参数为"Hello world!!"
    int pid = kernel_thread(init_main, "Hello world!!", 0);
    if (pid <= 0) {
        // 创建init_main线程失败
        panic("create init_main failed.\n");
    }

    // 获得initproc线程控制块
    initproc = find_proc(pid);
    // 设置initproc线程的名称
    set_proc_name(initproc, "init");

    assert(idleproc != NULL && idleproc->pid == 0);
    assert(initproc != NULL && initproc->pid == 1);
}
```

分配并初始化第0个内核线程idleproc之后，又通过kernel_thread函数创建了新的内核线程initproc，然后返回新线程的pid。

`int pid = kernel_thread(init_main, "Hello world!!", 0);`
kernel_thread的参数分别表示线程执行init_main函数，参数为"Hello world!!"，以及根据clone_flags决定是复制还是共享内存管理系统。但在lab4的copy_mm中并未实现内存管理系统的复制或共享。

在kernel_thread函数中，它构建了一个临时的中断栈帧tf，do_fork函数会调用copy_thread函数来在新创建的进程内核栈上专门给进程的中断帧分配一块空间。设置好tf的值之后，进入了do_fork函数。
#### kernel_thread:
```cpp
int 
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    // 对trameframe，也就是我们程序的一些上下文进行一些初始化
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));

    // 设置内核线程的参数和函数指针
    tf.gpr.s0 = (uintptr_t)fn; // s0 寄存器保存函数指针
    tf.gpr.s1 = (uintptr_t)arg; // s1 寄存器保存函数参数

    // 设置 trapframe 中的 status 寄存器（SSTATUS）
    // SSTATUS_SPP：Supervisor Previous Privilege（设置为 supervisor 模式，因为这是一个内核线程）
    // SSTATUS_SPIE：Supervisor Previous Interrupt Enable（设置为启用中断，因为这是一个内核线程）
    // SSTATUS_SIE：Supervisor Interrupt Enable（设置为禁用中断，因为我们不希望该线程被中断）
    tf.status = (read_csr(sstatus) | SSTATUS_SPP | SSTATUS_SPIE) & ~SSTATUS_SIE;

    // 将入口点（epc）设置为 kernel_thread_entry 函数，作用实际上是将pc指针指向它(*trapentry.S会用到)
    tf.epc = (uintptr_t)kernel_thread_entry;

    // 使用 do_fork 创建一个新进程（内核线程），这样才真正用设置的tf创建新进程。
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
```

do_fork函数任务如下：
1.分配并初始化进程控制块（alloc_proc函数）
2.分配并初始化内核栈（setup_stack函数）
3.根据clone_flags决定是复制还是共享内存管理系统（copy_mm函数）
4.设置进程的中断帧和上下文（copy_thread函数）
5.把设置好的进程加入链表
6.将新建的进程设为就绪态
7.将返回值设为线程id

如果前3步没有执行成功，都需要做出对应的出错处理，对操作进行回滚。
#### do_fork：
```cpp
//分配并初始化线程控制块
proc=alloc_proc();
if(proc==NULL){
    goto fork_out;    
}
proc->parent=current;
//分配并初始化内核栈
if(setup_kstack(proc)!=0){
    goto bad_fork_cleanup_proc;
}
//根据clone_flags决定是复制还是共享内存管理系统
if(cppy_mm(clone_flags,proc)!=0){
    goto bad_fork_cleanup_kstack;
}
//设置进程的中断帧和上下文
copy_thread(proc, stack, tf);
//把设置好的进程加入链表
bool intr_flag;
local_intr_save(intr_flag);//禁用中断，保证执行关键代码时不会被打扰
{
    proc->pid=get_pid();
    hash_proc(proc);
    list_add(&proc_list,&(proc->list_link));
    nr_process++;
}
local_intr_save(intr_flag);//恢复中断
//将新建的进程设为就绪态
wakeup_proc(proc);
//将返回值设为线程id
ret=proc->pid;
```

在其中的copy_thread函数中，首先在上面分配的内核栈上分配出一片空间来保存trapframe。然后，我们将trapframe中的a0寄存器（返回值）设置为0，说明这个进程是一个子进程。之后我们将上下文中的ra设置为了forkret函数的入口，并且把trapframe放在上下文的栈顶，为之后的进程切换打好基础。

#### copy_thread:
```cpp
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE - sizeof(struct trapframe));
    *(proc->tf) = *tf;

    // Set a0 to 0 so a child process knows it's just forked
    proc->tf->gpr.a0 = 0;
    proc->tf->gpr.sp = (esp == 0) ? (uintptr_t)proc->tf : esp;

    proc->context.ra = (uintptr_t)forkret;
    proc->context.sp = (uintptr_t)(proc->tf);
}
```

#### 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
分配pid由get_pid函数实现。在调用get_pid时，通过屏蔽中断的方式实现了get_pid的互斥，一次仅实现一个pid的生成。

## 练习3：编写proc_run 函数（需要编码）

proc_run主要过程如下：

1. 检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。
2. 使用 local_intr_save 宏禁用中断，保证在切换过程中不被中断打断。
3. 将当前进程指针 current 指向要运行的进程 proc。
4. 使用 lcr3 函数切换页表，以便使用新进程的地址空间。
5. 调用 switch_to 函数进行上下文切换，将当前进程的上下文切换为要运行的进程的上下文。
6. 使用 local_intr_restore 宏允许中断，恢复中断状态。

```cpp
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        // LAB4:EXERCISE3 你的代码
        /*
        * 一些有用的宏、函数和定义，你可以在下面的实现中使用它们。
        * 宏或函数：
        *   local_intr_save():        禁用中断
        *   local_intr_restore():     启用中断
        *   lcr3():                   修改 CR3 寄存器的值
        *   switch_to():              在两个进程之间进行上下文切换
        */
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        // 禁用中断，保存中断状态
        local_intr_save(intr_flag);
        // 设置当前运行的进程为指定的进程
        current = proc;
        // 加载新进程的页表基址
        lcr3(next->cr3);
        // 执行上下文切换，将从 prev 进程切换到 next 进程
        switch_to(&(prev->context), &(next->context));
        // 恢复之前保存的中断状态，启用中断
        local_intr_restore(intr_flag);
    }
}
```

执行switch_to后，会跳转到/kern/process/switch.S的switch_to汇编语句中，进行完成寄存器的保存和加载之后，跳到ret。而ret为copy_thread中设置的forkret。fortret函数执行forkrets函数，参数为当前线程的中断帧。forkrets在kern/trap/trapentry.S中，将sp赋值为中断帧之后，执行__trapret，然后跳转到sret。sret为kernel_thread中设置的kernel_thread_entry。它在/kern/process/entry.S中定义。执行s0（即线程主函数）后，执行do_exit函数。do_exit函数调用panic语句输出信息随后停止ucore。

#### 在本实验的执行过程中，创建且运行了几个内核线程？
在本实验的执行过程中，在proc_init中创建且运行了2个内核线程，分别是idle和init进程

```cpp
// 创建0号进程

    if ((idleproc = alloc_proc()) == NULL) {
        panic("cannot alloc idleproc.\n");
    }

...

// 创建1号进程

    current = idleproc;

    int pid = kernel_thread(init_main, "Hello world!!", 0);
    if (pid <= 0) {
        panic("create init_main failed.\n");
    }

    initproc = find_proc(pid);
    set_proc_name(initproc, "init");
```

## Challenge
#### 说明语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);是如何实现开关中断的？

相应代码如下。local_intr_save调用了__intr_save函数，通过判断当前寄存器状态是否为可中断来进行对应操作，然后返回相应布尔值，如果可中断就通过disable函数清空状态位。local_intr_save之所以通过`do{} while(0)`语句执行，是因为防止在调用宏定义是出现两个分号。

local_intr_restore调用了__intr_restore函数，通过flag的值决定是否设置中断状态。

```cpp
#define local_intr_save(x) \
    do {                   \
        x = __intr_save(); \
    } while (0)

static inline bool __intr_save(void) {
    if (read_csr(sstatus) & SSTATUS_SIE) {
        intr_disable();
        return 1;
    }
    return 0;
}

#define local_intr_restore(x) __intr_restore(x);

static inline void __intr_restore(bool flag) {
    if (flag) {
        intr_enable();
    }
}
```

#### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）。列出你认为OS原理中很重要，但在实验中没有对应上的知识点。

本实验中重要的知识点就是进程的初始化和切换，对应OS原理中的进程有关知识，通过switch_to对进程的上下文进行切换。在本实验中涉及的都是内核线程，并没有涉及用户态的线程或进程。