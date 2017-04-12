# OS lab4 Report

苏克

2014011402

## 练习0

通过meld将lab3中有用的代码转进lab4中即可。

## 练习1

### 实现

练习1要求分配并初始化一个新进程的进程控制块。所要初始化的值很多，但大多数只要置0即可，因为在创建之后还会有对其值赋值的工作，这里不需要进行。注意pid需要设置为-1而非0，这是实验指引“创建第0个内核线程”中所指出的。

在lab4中我们创建的都是内核线程，因而proc->cr3应当被设置为boot_cr3，即内核虚拟空间PDE的地址。

所要实现的alloc_proc代码如下：

```c
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 2014011402
    /*
     * 注释略去
     */
      proc->state = PROC_UNINIT;
      proc->pid = -1; // uninited
      proc->runs = 0;
      proc->kstack = 0;
      proc->need_resched = 0; // false
      proc->parent = 0; // null
      proc->mm = 0;
      memset(&(proc->context), 0, sizeof(proc->context));
      proc->tf = 0;
      proc->cr3 = boot_cr3;
      proc->flags = 0;
      memset(proc->name, 0, PROC_NAME_LEN);

    }
    return proc;
}
```

### proc_struct的两个成员变量context和tf的含义

context为进程的上下文，用于进程切换。在switch.S中，进程切换之前，将esp、ebp等寄存器的值都存入了原进程（from）的context中，并从后进程（to）的context中还原这些寄存器的值。

tf是中断帧的指针，指向内核栈的某个为止，记录进程从用户空间跳到内核空间时被中断之前的状态。当进程跳回用户空间时，需要调整中断帧来恢复寄存器的值。

## 练习2

### 实现

练习2要求我们实现do_fork函数。根据注释的指引，调用相关的函数即可完成该函数，代码如下（每一步注释都对应着一些操作）：

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    //LAB4:EXERCISE2 2014011402
    /*
     * 注释略去
     */

    //    1. call alloc_proc to allocate a proc_struct
    proc = alloc_proc();
    proc->parent = current;
    proc->pid = get_pid();
    //    2. call setup_kstack to allocate a kernel stack for child process
    setup_kstack(proc);
    //    3. call copy_mm to dup OR share mm according clone_flag
    copy_mm(clone_flags, proc);
    //    4. call copy_thread to setup tf & context in proc_struct
    copy_thread(proc, stack, tf);
    //    5. insert proc_struct into hash_list && proc_list
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    nr_process++;
    //    6. call wakeup_proc to make the new child process RUNNABLE
    wakeup_proc(proc);
    //    7. set ret vaule using child proc's pid
    ret = proc->pid;
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

### 实现的不足

lab答案中，对于一些有错误返回的操作，都进行了错误返回的判断和处理，增强了安全性。当然，在不发生错误的情况下，这两者并无差别。

### 每个新fork的线程是否能得到唯一id

可以得到唯一的pid，只要没有整数溢出（这简直是不可能的）。在lab4的答案中，分配pid的过程被保护锁保护住，在其中不允许中断，也就保证了pid的唯一性。这也是我的答案稍欠考虑的地方。

## 练习3

### proc_run 功能

proc_run代码如下：

```c
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```

可以看到，首先设置“转出”的进程prev为current，“转到”的进程next为proc，即函数的参数。在转入（current=proc）之后，首先从内核栈上读出esp0的值，然后从进程控制块中读出cr3的值，最后调用switch_to函数（其功能在上面已经提到）将各个寄存器的值还原。这样便完成了进程切换的过程。

### 创建了几个内核进程

从proc_init函数中可以看到，一共创建了两个进程。一个为idleproc。另一个为initproc。根据实验导引，前者完成各个子系统的初始化，后者则完成实验的一些输出，Hello Goodbye之类的。

### local_intr_* 的含义

保证在进程切换时不能发生中断。

## 总结

本次实验内容不多，但是涉及了一个内核进程从创建（初始化）到执行（或者说切换）的全过程，令我对进程的维护和控制过程更加熟悉。唯一令我惊讶的是c竟然不能用false和true为bool变量赋值，只能用0和1，这个事情确实是第一次见。

感谢助教老师的辛勤工作。
