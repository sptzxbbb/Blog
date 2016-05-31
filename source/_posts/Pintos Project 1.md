---
title: Pintos Project1
tags: [project]
category: Coding 
date: 2015-08-12
---

### 前言 ###
写这个Project教程有两个目的

1. 基于过往经验，写教程，不如说是记录我完成Project的整个流程，可以理顺整个人的思路，有助于理解和完成Project。
2. 在国内实在找不到像样的教程，只能自己写一个。

<font color="Red">如需转载，请注明出处，附上本博客链接，谢谢。</font>

实验代码放在Github上，提供参考。

<!--more-->

---

### Pintos实验简介 ###
Pintos是一個Standford的CS140的<font color="Red">操作系統教學Project</font>，以下簡介摘译自Standford的[Project官网](http://www.stanford.edu/class/cs140/projects/pintos/pintos.html#SEC_Contents)  

> Pintos是一个基于80x86架构的简单的操作系统框架。它支持内核线程、加载和运行用户程序、和一个文件系统。但它们的实现都十分简陋。在Pintos的Project里你和你的Project团队要增强这些领域的支持。你也需要增加虚拟内存的实现。

Pintos这个操作系统实际需要在system simulator里运行，本教学中我们提供**[Bochs](http://bochs.sourceforge.net/)**和**[QEMU](http://www.qemu.org/index.html)**两个模拟器。本Project中我选择用**Bochs**。

至于Pintos和Bochs的**环境安装配置**，  
1. [Pintos安装Standford指南](http://www.stanford.edu/class/cs140/projects/pintos/pintos_12.html#SEC166)  
2. 国内还有很多关于Pintos和Bochs的安装教程，例如[这个](http://www.qszxin.com/bowen/caozuoxitong/2014-10-16/24.html) 

<u>实验的正确性检验需要在Bochs上运行Pintos的Test样例，比对标准答案，判断Pintos实现是否正确。这一部分稍后再详述。</u>

整个Pintos实验分为四个部分，如下所示：

+ Project1: Thread
+ Project2: User Programs
+ Project3: Virtual Memory
+ Project4: File System

---

### **Project1: Thread** ###

*Projcet1*的目标是要对**多线程**提供更可靠的支持，具体有如下的四点要求。

+ Alarm Clock(Non-busy sleep)
+ Priority Scheduler(Priority scheduling, Priority donation)
+ Advanced Scheduler(BSD-style scheduler)

整个Project1的工作都会在`src/threads/` & `src/devices/`下完成。

下面分为四个部分展开

### Alarm Clock ###
需要我们修改的函数是src/devices/timer.c里面的`time_sleep()`, 来避免忙等待。函数如下。

~~~c++
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks)
{
  int64_t start = timer_ticks ();

  ASSERT (intr_get_level () == INTR_ON);

  while (timer_elapsed (start) < ticks)
    thread_yield ();

}
~~~

整个函数首先调用了`timer_ticks()`，它返回了Pintos启动到当前的时间，实现如下。

~~~c++
/* Returns the number of timer ticks since the OS booted. */
int64_t
timer_ticks (void)
{
  enum intr_level old_level = intr_disable ();
  int64_t t = ticks;
  intr_set_level (old_level);
  return t;
}
~~~

其首先定义了一个enum枚举量`old_level`,在src/threads/interrupt.h里可以找到它的定义。从定义来看，是一个用于表示中断开与闭的枚举类型。

~~~c++
/* Interrupts on or off? */
enum intr_level 
  {
    INTR_OFF,             /* Interrupts disabled. */
    INTR_ON               /* Interrupts enabled. */
  };
~~~

`old_level`接收了`intr_disable()`的返回值，其定义如下

~~~c++
/* Disables interrupts and returns the previous interrupt status. */
enum intr_level
intr_disable (void) 
{
  enum intr_level old_level = intr_get_level ();

  /* Disable interrupts by clearing the interrupt flag.
     See [IA32-v2b] "CLI" and [IA32-v3a] 5.8.1 "Masking Maskable
     Hardware Interrupts". */
  asm volatile ("cli" : : : "memory");

  return old_level;
}
~~~

它嵌套调用了`intr_get_level()`, 其定义如下

~~~c++
/* Returns the current interrupt status. */
enum intr_level
intr_get_level (void) 
{
  uint32_t flags;

  /* Push the flags register on the processor stack, then pop the
     value off the stack into `flags'.  See [IA32-v2b] "PUSHF"
     and "POP" and [IA32-v3a] 5.8.1 "Masking Maskable Hardware
     Interrupts". */
  asm volatile ("pushfl; popl %0" : "=g" (flags));

  return flags & FLAG_IF ? INTR_ON : INTR_OFF;
}
~~~

它就是一系列中断函数调用的终点，我们慢慢来解剖这个函数。  
asm表示c内嵌汇编指令的开始，关键字volatile阻止gcc对此内联汇编优化(实现原子操作关键的一步，gcc的优化往往导致内存屏障)，整条指令将EFLAGS寄存器的值压栈，在出栈存入`flags`变量当中(g参数表示操作数可以是内存变量，立即数，EAX,EBX,ECX或者EDX)。因此执行后`flags`是由若干标识位构成的值，然后和`FLAG_IF`进行按位与操作，目的是判断中断标志位的值。`FLAG_IF`在`flags.h`定义如下，

~~~c++
#ifndef THREADS_FLAGS_H
#define THREADS_FLAGS_H

/* EFLAGS Register. */
#define FLAG_MBS  0x00000002    /* Must be set. */
#define FLAG_IF   0x00000200    /* Interrupt Flag. */

#endif /* threads/flags.h */
~~~

到了这里已经十分清晰，`intr_get_level()`是用于返回一个表示中断标志位状态的枚举量。再往上一层，对于调用它的`intr_disable()`，**通过CLI指令置中断标志位IF为0,屏蔽中断信号**，然后把执行CLI之前，CPU原来的中断状态进行返回。

~~~c++
asm volatile ("cli" : : : "memory");
~~~

接下来程序回到`timer_ticks()`，紧接着引用了一个全局变量`ticks`，注意它与`timer_sleep(int64_t ticks)`接受的参数同名，但函数内部引用的`ticks`是参数传进来的`ticks`,全局变量`ticks`定义如下

~~~c++
/* Number of timer ticks since OS booted. */
static int64_t ticks;
~~~

下一步调用了`intr_set_level()`,定义如下。

~~~c++
/* Enables or disables interrupts as specified by LEVEL and
   returns the previous interrupt status. */
enum intr_level
intr_set_level (enum intr_level level) 
{
  return level == INTR_ON ? intr_enable () : intr_disable ();
}
~~~

里面嵌套调用了`intr_enable()`和`intr_disable()`，后者我们已讨论过，至于前者定义如下。

~~~c++
/* Enables interrupts and returns the previous interrupt status. */
enum intr_level
intr_enable (void) 
{
  enum intr_level old_level = intr_get_level ();
  ASSERT (!intr_context ());

  /* Enable interrupts by setting the interrupt flag.

     See [IA32-v2b] "STI" and [IA32-v3a] 5.8.1 "Masking Maskable
     Hardware Interrupts". */
  asm volatile ("sti");
  return old_level;
}
~~~

`intr_enable()`调用了一个新的函数`intr_context()`，定义如下。

~~~c++
/* Returns true during processing of an external interrupt
   and false at all other times. */
bool
intr_context (void) 
{
  return in_external_intr;
}
~~~

再附上`in_external_intr`的声明和定义。

~~~c++
/* External interrupts are those generated by devices outside the
   CPU, such as the timer.  External interrupts run with
   interrupts turned off, so they never nest, nor are they ever
   pre-empted.  Handlers for external interrupts also may not
   sleep, although they may invoke intr_yield_on_return() to
   request that a new process be scheduled just before the
   interrupt returns. */
static bool in_external_intr;   /* Are we processing an external interrupt? */
~~~

从上述comment可知，`in_external_intr`是一个判断是否有外部中断的bool变量。再回到上一层，`intr_context()`是一个判断当前是否在处理外部中断的函数。再回到上一层，`intr_enable()`利用断言和`intr_context()`判断当前不是处理外部中断的话，打开中断标志位，再返回旧的中断标志位。


再往上一层，`intr_set_level`函数的功能便十分明显，按照参数`level`设置中断位的状态。`timer_ticks()`调用它是为了使中断位恢复调用`intr_disable()`前的状态。  
到了这里，`timer_ticks()`执行结束，这一系列的函数调用显然是为了：**<font color="Red">使这个函数原子化，不能被中断，否则执行t=ticks之前可能因为中断产生延迟，继而t变得不准确</font>**。

我们对`timer_ticks()`的研究告一段落，接下来程序的执行回到最初调用者`timer_sleep()`。

---

下一步判断了中断标识位是否打开，中断必须打开的原因是`timer_sleep`马上就要进入忙等待，而如果关闭了中断，它将跑完整个loop或者直到time slice耗尽。<font color="Red">这极大降低了CPU效率，因此要避免发生，虽然忙等待本身就十分低效。</font>接下来我们研究`timer_sleep()`的忙等待具体是如何实现。

接下来调用的第一个函数`timer_elapsed()`定义如下，实现十分简单，返回调用时刻与参数`then`的时间差。

~~~c++
/* Returns the number of timer ticks elapsed since THEN, which
   should be a value once returned by timer_ticks(). */
int64_t
timer_elapsed (int64_t then)
{
  return timer_ticks () - then;
}
~~~

那么`timer_sleep()`的函数功能已经变得十分明朗：从执行本routine时刻起的`ticks`的时间内，一直循环调用`thread_yiele()`。显然这个函数是忙等待的关键，从名字上来看，让出CPU。定义如下。

~~~c++
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void
thread_yield (void) 
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;

  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread) 
    list_push_back (&ready_list, &cur->elem);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
~~~

从这里开始涉及到thread，线程，结构定义如下。

~~~c++
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */
  };
~~~

值得注意的地方有几个，首先tid_t是一个typedef，具体是int。

~~~c++
/* Thread identifier type.
   You can redefine this to whatever type you like. */
typedef int tid_t;
~~~

第二个变量是枚举变量，表示thread的状态。  

~~~c++
/* States in a thread's life cycle. */
enum thread_status
  {
    THREAD_RUNNING,     /* Running thread. */
    THREAD_READY,       /* Not running but ready to run. */
    THREAD_BLOCKED,     /* Waiting for an event to trigger. */
    THREAD_DYING        /* About to be destroyed. */
  };

~~~

第五个变量`priority`是线程优先级，共有64级，定义如下

~~~c++
/* Thread priorities. */
#define PRI_MIN 0                       /* Lowest priority. */
#define PRI_DEFAULT 31                  /* Default priority. */
#define PRI_MAX 63                      /* Highest priority. */
~~~

然后是`list_elem`这个结构体，定义位于`/src/lib/kernel/list.h`，具体如下。

~~~c++
/* List element. */
struct list_elem 
  {
    struct list_elem *prev;     /* Previous list element. */
    struct list_elem *next;     /* Next list element. */
  };
~~~
  
其次是`pagedir`，用于定位虚拟内存页表的变量，只在USERPROG下有效。  
最后是神奇的`magic`。定义如下：

~~~c++
/* Random value for struct thread's `magic' member.
   Used to detect stack overflow.  See the big comment at the top
   of thread.h for details. */
#define THREAD_MAGIC 0xcd6abf4b
~~~

了解完`thread`的结构之后，我们来看看`thread_current()`的具体实现。

~~~c++
/* Returns the running thread.
   This is running_thread() plus a couple of sanity checks.
   See the big comment at the top of thread.h for details. */
struct thread *
thread_current (void) 
{
  struct thread *t = running_thread ();
  
  /* Make sure T is really a thread.
     If either of these assertions fire, then your thread may
     have overflowed its stack.  Each thread has less than 4 kB
     of stack, so a few big automatic arrays or moderate
     recursion can cause stack overflow. */
  ASSERT (is_thread (t));
  ASSERT (t->status == THREAD_RUNNING);

  return t;
}
~~~

里面嵌套调用了`running_thread()`和`is_thread()`，实现如下。

~~~c++
/* Returns true if T appears to point to a valid thread. */
static bool
is_thread (struct thread *t)
{
  return t != NULL && t->magic == THREAD_MAGIC;
}
~~~

判断是否t不空以及`t->magic`与THREAD_MAGIC相等(**stack overflow**时通常会改变magic值。)

~~~c++
/* Returns the running thread. */
struct thread *
running_thread (void) 
{
  uint32_t *esp;

  /* Copy the CPU's stack pointer into `esp', and then round that
     down to the start of a page.  Because `struct thread' is
     always at the beginning of a page and the stack pointer is
     somewhere in the middle, this locates the curent thread. */
  asm ("mov %%esp, %0" : "=g" (esp));
  return pg_round_down (esp);
}
~~~

本函数功能是返回当前运行的线程页面的首地址，具体做法是把`esp寄存器`堆栈指针的值取出赋给指针`esp`，对其调用`pg_round_down()`进行取整，得到线程页面的首地址。后者位于src/threads/vaddr.h，定义如下。

~~~c++
/* Round down to nearest page boundary. */
static inline void *pg_round_down (const void *va) {
  return (void *) ((uintptr_t) va & ~PGMASK);
}
~~~

其中相关的代码如下

~~~c++
typedef uint32_t uintptr_t;
#define UINTPTR_MAX UINT32_MAX
~~~

~~~c++
/* Functions and macros for working with virtual addresses.

   See pte.h for functions and macros specifically for x86
   hardware page tables. */

#define BITMASK(SHIFT, CNT) (((1ul << (CNT)) - 1) << (SHIFT))

/* Page offset (bits 0:12). */
#define PGSHIFT 0                          /* Index of first offset bit. */
#define PGBITS  12                         /* Number of offset bits. */
#define PGSIZE  (1 << PGBITS)              /* Bytes in a page. */
#define PGMASK  BITMASK(PGSHIFT, PGBITS)   /* Page offset bits (0:12). */
~~~

这里需要用到页面的知识，Pintos里一个页面为4KB = 4 × 2<sup>10</sup> bits = 2<sup>12</sup> bits.即以12位的大小可以表示一个页面。对一个地址低12位清零后，可以得到其页面的起始地址。具体计算过程请结合代码参考下图。

~~~c++
sptzxb $ ./main
(1ul << (CNT)) << (SHIFT)    :1000
(1ul << (CNT)) - 1 << (SHIFT):fff
~PGMASK                      :fffff000
~~~

返回到`thread_current()`，其功能是判断当前运行的进程是否一个有效的进程，如果有效便返回。明白其功能后，我们再返回到上一层`thread_yield()`，我们知道cur指向了一个正在运行的有效线程。如果cur并非是`idle_thread`，把其加入到`ready_list`当中同时相应调整其状态，最后进行调度`schedule()`，整个过程保持原子性。相关代码如下。

~~~c++
/* Idle thread. */
static struct thread *idle_thread;
~~~

~~~
/* Inserts ELEM at the end of LIST, so that it becomes the
   back in LIST. */
void
list_push_back (struct list *list, struct list_elem *elem)
{
  list_insert (list_end (list), elem);
}
~~~

~~~c++
/* Inserts ELEM just before BEFORE, which may be either an
   interior element or a tail.  The latter case is equivalent to
   list_push_back(). */
void
list_insert (struct list_elem *before, struct list_elem *elem)
{
  ASSERT (is_interior (before) || is_tail (before));
  ASSERT (elem != NULL);

  elem->prev = before->prev;
  elem->next = before;
  before->prev->next = elem;
  before->prev = elem;
}
~~~

关键代码`schedule()`单独列出。

~~~c++
/* Schedules a new process.  At entry, interrupts must be off and
   the running process's state must have been changed from
   running to some other state.  This function finds another
   thread to run and switches to it.

   It's not safe to call printf() until thread_schedule_tail()
   has completed. */
static void
schedule (void) 
{
  struct thread *cur = running_thread ();
  struct thread *next = next_thread_to_run ();
  struct thread *prev = NULL;

  ASSERT (intr_get_level () == INTR_OFF);
  ASSERT (cur->status != THREAD_RUNNING);
  ASSERT (is_thread (next));

  if (cur != next)
    prev = switch_threads (cur, next);
  thread_schedule_tail (prev);
}
~~~

想读懂这个函数，需要了解`next_thread_to_run()`，`switch_threads()`和`thread_schedule_tail()`，代码如下。

~~~c++
/* Chooses and returns the next thread to be scheduled.  Should
   return a thread from the run queue, unless the run queue is
   empty.  (If the running thread can continue running, then it
   will be in the run queue.)  If the run queue is empty, return
   idle_thread. */
static struct thread *
next_thread_to_run (void) 
{
  if (list_empty (&ready_list))
    return idle_thread;
  else
    return list_entry (list_pop_front (&ready_list), struct thread, elem);
}
~~~

简单来说，如果当前就绪队列为空，则返回一个`idle_thread`，否则返回就绪队列中第一个线程。

至于`switch_threads()`声明如下。

~~~c++
/* Switches from CUR, which must be the running thread, to NEXT,
   which must also be running switch_threads(), returning CUR in
   NEXT's context. */
struct thread *switch_threads (struct thread *cur, struct thread *next);
~~~

具体是以汇编实现。

~~~
.globl switch_threads
.func switch_threads
switch_threads:
	# Save caller's register state.
	#
	# Note that the SVR4 ABI allows us to destroy %eax, %ecx, %edx,
	# but requires us to preserve %ebx, %ebp, %esi, %edi.  See
	# [SysV-ABI-386] pages 3-11 and 3-12 for details.
	#
	# This stack frame must match the one set up by thread_create()
	# in size.
	pushl %ebx
	pushl %ebp
	pushl %esi
	pushl %edi

	# Get offsetof (struct thread, stack).
.globl thread_stack_ofs
	mov thread_stack_ofs, %edx

	# Save current stack pointer to old thread's stack, if any.
	movl SWITCH_CUR(%esp), %eax
	movl %esp, (%eax,%edx,1)

	# Restore stack pointer from new thread's stack.
	movl SWITCH_NEXT(%esp), %ecx
	movl (%ecx,%edx,1), %esp

	# Restore caller's register state.
	popl %edi
	popl %esi
	popl %ebp
	popl %ebx
        ret
.endfunc
~~~

保存当前线程的ebx,ebp,esi,edi和esp,恢复新线程的ebx,ebp,esi,esp.

下一步，调用`thread_schedule_tial`清零时间片，并切换到新的页面，并且销毁旧页面，如果上一线程进入了DYING。

~~~c++
/* Completes a thread switch by activating the new thread's page
   tables, and, if the previous thread is dying, destroying it.

   At this function's invocation, we just switched from thread
   PREV, the new thread is already running, and interrupts are
   still disabled.  This function is normally invoked by
   thread_schedule() as its final action before returning, but
   the first time a thread is scheduled it is called by
   switch_entry() (see switch.S).

   It's not safe to call printf() until the thread switch is
   complete.  In practice that means that printf()s should be
   added at the end of the function.

   After this function and its caller returns, the thread switch
   is complete. */
void
thread_schedule_tail (struct thread *prev)
{
  struct thread *cur = running_thread ();
  
  ASSERT (intr_get_level () == INTR_OFF);

  /* Mark us as running. */
  cur->status = THREAD_RUNNING;

  /* Start new time slice. */
  thread_ticks = 0;

#ifdef USERPROG
  /* Activate the new address space. */
  process_activate ();
#endif

  /* If the thread we switched from is dying, destroy its struct
     thread.  This must happen late so that thread_exit() doesn't
     pull out the rug under itself.  (We don't free
     initial_thread because its memory was not obtained via
     palloc().) */
  if (prev != NULL && prev->status == THREAD_DYING && prev != initial_thread) 
    {
      ASSERT (prev != cur);
      palloc_free_page (prev);
    }
}
~~~

切换新页面时调用了`process_activate`。

~~~c++
void
process_activate (void)
{
  struct thread *t = thread_current ();

  /* Activate thread's page tables. */
  pagedir_activate (t->pagedir);

  /* Set thread's kernel stack for use in processing
     interrupts. */
  tss_update ();
}
~~~

激活新页面又需要调用`pagedir_activate()`，这个函数更新了页目录表，与linux实现相似。

~~~c++
/* Loads page directory PD into the CPU's page directory base
   register. */
void
pagedir_activate (uint32_t *pd) 
{
  if (pd == NULL)
    pd = init_page_dir;

  /* Store the physical address of the page directory into CR3
     aka PDBR (page directory base register).  This activates our
     new page tables immediately.  See [IA32-v2a] "MOV--Move
     to/from Control Registers" and [IA32-v3a] 3.7.5 "Base
     Address of the Page Directory". */
  asm volatile ("movl %0, %%cr3" : : "r" (vtop (pd)) : "memory");
}
~~~

`process_activate()`下一步调用了`tss_update()`。

~~~c++
/* Sets the ring 0 stack pointer in the TSS to point to the end
   of the thread stack. */
void
tss_update (void) 
{
  ASSERT (tss != NULL);
  tss->esp0 = (uint8_t *) thread_current () + PGSIZE;
}
~~~

TSS是任务切换时，用于保存任务现场各寄存器状态的内存块。CPU把即将挂起的任务的寄存器的值写回到该任务的TSS中，再从马上恢复的任务的TSS中读取寄存器的值写回到CPU当中，完成任务切换。**从上面代码来看，Pintos并没有真正利用TSS，只是象征性的简单更新了线程的堆栈底部地址。**这样看来，TSS在Pintos里就是个摆设。是的，Pintos自己也承认了。既然如此，那么TSS没有太大的研究意义。

> Instances of the TSS, an x86-specific structure, are used to define "tasks", a form of support for multitasking built right into the processor.  However, for various reasons including portability, speed, and flexibility, **<font color="Red">most x86 OSes almost completely ignore the TSS.We are no exception</font>**.

`thread_schedule_tail()`里最后调用的`palloc_free_page()`，完成的是线程页面的销毁工作。

~~~c++
/* Frees the page at PAGE. */
void
palloc_free_page (void *page) 
{
  palloc_free_multiple (page, 1);
}
~~~

间接调用了`pallpc_free_multiple()`，释放从`prev`开始的1个页面.

~~~c++
/* Frees the PAGE_CNT pages starting at PAGES. */
void
palloc_free_multiple (void *pages, size_t page_cnt) 
{
  struct pool *pool;
  size_t page_idx;

  ASSERT (pg_ofs (pages) == 0);
  if (pages == NULL || page_cnt == 0)
    return;

  if (page_from_pool (&kernel_pool, pages))
    pool = &kernel_pool;
  else if (page_from_pool (&user_pool, pages))
    pool = &user_pool;
  else
    NOT_REACHED ();

  page_idx = pg_no (pages) - pg_no (pool->base);

#ifndef NDEBUG
  memset (pages, 0xcc, PGSIZE * page_cnt);
#endif

  ASSERT (bitmap_all (pool->used_map, page_idx, page_cnt));
  bitmap_set_multiple (pool->used_map, page_idx, page_cnt, false);
}
~~~
引进了一个新的结构体`pool`，当然还有两个结构体成员`bitmap`和`lock`，我们一一分析。

~~~c++
/* A memory pool. */
struct pool
  {
    struct lock lock;                   /* Mutual exclusion. */
    struct bitmap *used_map;            /* Bitmap of free pages. */
    uint8_t *base;                      /* Base of pool. */
  };
~~~
  
Pintos系统的内存分为两个`pool`，一个是用户池，用于虚拟内存页面，另一个是内核池，用于其他事务。默认下，内存平均分成两部分给两个池使用。

锁`lock`结构体由一个指向锁拥有者的线程指针和一个信号量组成。

~~~c++
/* Lock. */
struct lock 
  {
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
  };
~~~
  
至于信号量，由当前的资源数目和当前的等待队列组成，其定义如下

~~~c++
/* A counting semaphore. */
struct semaphore 
  {
    unsigned value;             /* Current value. */
    struct list waiters;        /* List of waiting threads. */
  };
~~~

位图`bitmap`的结构体实际上是bit组成的数组。

~~~c++
/* Element type.

   This must be an unsigned integer type at least as wide as int.

   Each bit represents one bit in the bitmap.
   If bit 0 in an element represents bit K in the bitmap,
   then bit 1 in the element represents bit K+1 in the bitmap,
   and so on. */
typedef unsigned long elem_type;

/* From the outside, a bitmap is an array of bits.  From the
   inside, it's an array of elem_type (defined above) that
   simulates an array of bits. */
struct bitmap
  {
    size_t bit_cnt;     /* Number of bits. */
    elem_type *bits;    /* Elements that represent bits. */
  };
~~~

好，我们继续往下看，`pg_ofs()`被调用

~~~c++
/* Offset within a page. */
static inline unsigned pg_ofs (const void *va) {
  return (uintptr_t) va & PGMASK;
}
~~~

返回指针在页面内的位移。

断言判断为0,即是页面的起始地址。

~~~c++
  ASSERT (pg_ofs (pages) == 0);
~~~

然后判断需要释放的线程指针与页面的数目。

~~~c++
if (pages == NULL || page_cnt == 0)
    return;
~~~

从逻辑上来看，下一步判断需要释放的页面是来自内核池还是用户池。

~~~c++
  if (page_from_pool (&kernel_pool, pages))
    pool = &kernel_pool;
  else if (page_from_pool (&user_pool, pages))
    pool = &user_pool;
  else
    NOT_REACHED ();
~~~

里面调用了`page_from_pool()`来判断是否来自内核池与用户池。

~~~c++
/* Returns true if PAGE was allocated from POOL,
   false otherwise. */
static bool
page_from_pool (const struct pool *pool, void *page) 
{
  size_t page_no = pg_no (page);
  size_t start_page = pg_no (pool->base);
  size_t end_page = start_page + bitmap_size (pool->used_map);

  return page_no >= start_page && page_no < end_page;
}
~~~

具体实现用到了`pg_no`，即page number，页面数目， 对页面地址右移页面大小的长度即可得到。

~~~c++
/* Virtual page number. */
static inline uintptr_t pg_no (const void *va) {
  return (uintptr_t) va >> PGBITS;
}
~~~

还用到了`bitmap_size()`函数返回用过的页面数目，如下。

~~~c++
/* Bitmap size. */

/* Returns the number of bits in B. */
size_t
bitmap_size (const struct bitmap *b)
{
  return b->bit_cnt;
}
~~~

当要释放的页面既不是来自用户池，也不是来自内核池，程序调用了`NOT_REACHED()`来处理这个错误。

~~~c++
#ifndef __LIB_DEBUG_H
#define __LIB_DEBUG_H

/* GCC lets us add "attributes" to functions, function
   parameters, etc. to indicate their properties.
   See the GCC manual for details. */
#define UNUSED __attribute__ ((unused))
#define NO_RETURN __attribute__ ((noreturn))
#define NO_INLINE __attribute__ ((noinline))
#define PRINTF_FORMAT(FMT, FIRST) __attribute__ ((format (printf, FMT, FIRST)))

/* Halts the OS, printing the source file name, line number, and
   function name, plus a user-specific message. */
#define PANIC(...) debug_panic (__FILE__, __LINE__, __func__, __VA_ARGS__)

void debug_panic (const char *file, int line, const char *function,
                  const char *message, ...) PRINTF_FORMAT (4, 5) NO_RETURN;
void debug_backtrace (void);
void debug_backtrace_all (void);

#endif



/* This is outside the header guard so that debug.h may be
   included multiple times with different settings of NDEBUG. */
#undef ASSERT
#undef NOT_REACHED

#ifndef NDEBUG
#define ASSERT(CONDITION)                                       \
        if (CONDITION) { } else {                               \
                PANIC ("assertion `%s' failed.", #CONDITION);   \
        }
#define NOT_REACHED() PANIC ("executed an unreachable statement");
#else
#define ASSERT(CONDITION) ((void) 0)
#define NOT_REACHED() for (;;)
#endif /* lib/debug.h */
~~~

**没有定义NDEBUG的情况下，NOT_REACHED()调用PANIC函数，输出文件，行数，函数，错误信息等。**  

<font color="Red">在定义了NDEBUG的情况下，NOT_REACHED()调用了一个死循环来挂起OS，也就是宕机，OS会动弹不得。</font>  

延伸阅读：  
[关于内核错误的相对处理](http://zh.wikipedia.org/wiki/%E5%86%85%E6%A0%B8%E9%94%99%E8%AF%AF)   

[GCC Attribute属性](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html) 

我们对`NOT_REACHED`的研究告一段落，接下来`palloc_free_multiple`调用了`pg_no`来<font color="Red">计算当前页面在池里的索引</font>`page_idx`。

~~~c++
  page_idx = pg_no (pages) - pg_no (pool->base);
~~~
接下来这段代码对该线程的**所有页面用0xcc来进行初始化**，<font color="Red">目的未明</font>。

~~~c++
#ifndef NDEBUG
  memset (pages, 0xcc, PGSIZE * page_cnt);
#endif
~~~

紧接着的断言里调用了`bitmap_all`来测试所有位都为True。

~~~c++
/* Returns true if every bit in B between START and START + CNT,
   exclusive, is set to true, and false otherwise. */
bool
bitmap_all (const struct bitmap *b, size_t start, size_t cnt) 
{
  return !bitmap_contains (b, start, cnt, false);
}
~~~

里面嵌套调用了`bitmap_contains()`，<font color="Red">任意一个bit为value则返回True，否则返回False。</font>

~~~c++
/* Returns true if any bits in B between START and START + CNT,
   exclusive, are set to VALUE, and false otherwise. */
bool
bitmap_contains (const struct bitmap *b, size_t start, size_t cnt, bool value) 
{
  size_t i;
  ASSERT (b != NULL);
  ASSERT (start <= b->bit_cnt);
  ASSERT (start + cnt <= b->bit_cnt);

  for (i = 0; i < cnt; i++)
    if (bitmap_test (b, start + i) == value)
      return true;
  return false;
}
~~~

首先断言了参数的合法性。然后调用`bitmap_test()`来观察b位图中[0, cnt)每一位的值。注意：<font color="Red">此处传入的参数idx是即将释放的线程的起始页面在对应池中的页面索引，即第几个页面相对于对应池的base页面</font>。

~~~c++
/* Returns the value of the bit numbered IDX in B. */
bool
bitmap_test (const struct bitmap *b, size_t idx) 
{
  ASSERT (b != NULL);
  ASSERT (idx < b->bit_cnt);
  return (b->bits[elem_idx (idx)] & bit_mask (idx)) != 0;
}
~~~

~~~c++
/* Returns the index of the element that contains the bit
   numbered BIT_IDX. */
static inline size_t
elem_idx (size_t bit_idx) 
{
  return bit_idx / ELEM_BITS;
}
~~~

~~~c++
/* Number of bits in an element. */
#define ELEM_BITS (sizeof (elem_type) * CHAR_BIT)

#define CHAR_BIT 8
~~~

~~~c++
/* Returns an elem_type where only the bit corresponding to
   BIT_IDX is turned on. */
static inline elem_type
bit_mask (size_t bit_idx) 
{
  return (elem_type) 1 << (bit_idx % ELEM_BITS);
}
~~~

<font color="Red">目前暂时还不理解上述对位图的判断有何意义。</font>

整个函数最后的调用是`bitmap_set_multiple()`。

~~~c++
  bitmap_set_multiple (pool->used_map, page_idx, page_cnt, false);
~~~
  
对应函数定义如下：

~~~c++
/* Sets the CNT bits starting at START in B to VALUE. */
void
bitmap_set_multiple (struct bitmap *b, size_t start, size_t cnt, bool value) 
{
  size_t i;
  
  ASSERT (b != NULL);
  ASSERT (start <= b->bit_cnt);
  ASSERT (start + cnt <= b->bit_cnt);

  for (i = 0; i < cnt; i++)
    bitmap_set (b, start + i, value);
}
~~~

首先断言参数合法性。然后调用`bitmap_set()`，value为1，设b位图idx位为true，否则设为false。

~~~c++
/* Atomically sets the bit numbered IDX in B to VALUE. */
void
bitmap_set (struct bitmap *b, size_t idx, bool value) 
{
  ASSERT (b != NULL);
  ASSERT (idx < b->bit_cnt);
  if (value)
    bitmap_mark (b, idx);
  else
    bitmap_reset (b, idx);
}
~~~

~~~c++
/* Atomically sets the bit numbered BIT_IDX in B to true. */
void
bitmap_mark (struct bitmap *b, size_t bit_idx) 
{
  size_t idx = elem_idx (bit_idx);
  elem_type mask = bit_mask (bit_idx);

  /* This is equivalent to `b->bits[idx] |= mask' except that it
     is guaranteed to be atomic on a uniprocessor machine.  See
     the description of the OR instruction in [IA32-v2b]. */
  asm ("orl %1, %0" : "=m" (b->bits[idx]) : "r" (mask) : "cc");
}

/* Atomically sets the bit numbered BIT_IDX in B to false. */
void
bitmap_reset (struct bitmap *b, size_t bit_idx) 
{
  size_t idx = elem_idx (bit_idx);
  elem_type mask = bit_mask (bit_idx);

  /* This is equivalent to `b->bits[idx] &= ~mask' except that it
     is guaranteed to be atomic on a uniprocessor machine.  See
     the description of the AND instruction in [IA32-v2a]. */
  asm ("andl %1, %0" : "=m" (b->bits[idx]) : "r" (~mask) : "cc");
}
~~~

分析到这里，`palloc_free_multiple()`的目的很明确，**<font color="Red">将需要释放的线程的页面的位图进行清0，表示该页面空闲，能够再次被分配使用。</font>**

从头整理`timer_sleep()`的调用过程，得到了如下的函数调用图(省略一干细节调用)。


~~~c++
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks)
{
  int64_t start = timer_ticks ();

  ASSERT (intr_get_level () == INTR_ON);

  while (timer_elapsed (start) < ticks)
    thread_yield ();

}
~~~

![function call](/images/2015-05-02-Pintos Project1/call.png) 

1. `timer_sleep()`首先调用了`timer_ticks()`记录程序开始的时间点。
2. 然后调用`timer_elapsed()`返回自调用开始经过的时间。
3. 若经过时间小于参数`ticks`，调用`thread_yield()`，完成调度工作。

`thread_yield()`把当前运行的线程状态置为READY，放入就绪队列的队尾，然后从就绪队列的头部取出一个队列，作为新的当前运行线程。

可以看出，Pintos此时的调度是[FCFS](http://en.wikipedia.org/wiki/First-come,_first-served)。

整个**忙等待**做的无用功在于就绪队列中每次轮到其运行时，检测逝去时间，不满足的话马上进行调度，而根据前面的分析我们知道，每次进行调度调用的函数众多，这造成比较大的overhead，因次避免忙等待即使避免频繁调度。

**<font color="Red">重新实现timer_sleep()，我的思路是把当前线程状态置为BLOCKED，并把它加入到sleep_list当中。thread结构体中增加一个wakeup_ticks，每一tick调用timer_interrupt()时候，对sleep队列中的所有线程进行检测，若当前ticks >= wakeup_ticks，则将该线程唤醒，状态置为READY，并加入到就绪队列当中。</font>**


重新实现后的`timer_sleep()`

~~~c++
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks)
{
    if (ticks <= 0)
    {
        return;
    }
    thread_sleep(ticks);
}
~~~

直接调用`thread_sleep()`

~~~c++
void
thread_sleep(int64_t ticks)
{
    enum intr_level old_level;
    struct thread* cur = thread_current();

    ASSERT(THREAD_RUNNING == cur->status);

    old_level = intr_disable ();
    cur->wakeup_ticks = ticks + timer_ticks ();
    list_insert_ordered (&sleep_list, &cur->elem, thread_cmp_wakeup_ticks, NULL);
    thread_block ();
    intr_set_level (old_level);
}
~~~

thread结构体新增一个成员`wakeup_ticks`，记录唤醒时间，然后插入有序队伍`sleep_list`，再调用`thread_block()`。

涉及到的比较函数`thread_cmp_wakeup_ticks`:

~~~c++
bool
thread_cmp_wakeup_ticks (struct list_elem *a, struct list_elem * b, void *aux)
{
    ASSERT (a != NULL);
    ASSERT (b != NULL);

    struct thread *A = list_entry(a, struct thread, elem);
    struct thread *B = list_entry(b, struct thread, elem);

    return A->wakeup_ticks < B->wakeup_ticks;
}
~~~

然后在每tick调用的`timer_interrupt()`中调用`thread_wakeup()`。

~~~c++
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
//  thread_foreach (thread_wakeup, 0);
  thread_wakeup();
  thread_tick ();
}
~~~

`thread_wakeup`扫描有序的sleep_list，将睡眠进程唤醒。当需要扫描到第一个t->wakeup_ticks > timer_ticks()时，扫描结束。

~~~c++
void
thread_wakeup(void)
{
  struct list_elem *cur; // Current element in the list
  struct list_elem *next; // Next element connected to the current one
  struct thread *t;
  enum intr_level old_level;

  if (list_empty (&sleep_list))
    return;

  cur = list_begin (&sleep_list);
  while (cur != list_end (&sleep_list))
    {
        next = list_next (cur);
        t = list_entry (cur, struct thread, elem);
        if (t->wakeup_ticks > timer_ticks())
        break;

        old_level = intr_disable ();
        list_remove(cur);
        thread_unblock(t);
        intr_set_level (old_level);

        cur = next;
    }
}
~~~

Alarm Clock实现到此完成。

![AlarmClock](/images/2015-05-02-Pintos Project1/alarm-clock.png) 

---

### Priority Scheduler ###


#### Priority Scheduling ####

进程调度在上一节有提到是简单的FCFS，thread中虽然有priority，但并无应用。要实现优先级调度的话，只需对ready_list中的线程依据优先级排序。

构造比较函数`thread_cmp_priority`

~~~c++
bool
thread_cmp_priority (struct list_elem *a, struct list_elem *b, void *aux)
{
    ASSERT (a != NULL && b != NULL);

    struct thread *A = list_entry(a, struct thread, elem);
    struct thread *B = list_entry(b, struct thread, elem);

    return A->priority > B->priority;
}
~~~

然后把`thread_unblock()`，`thread_yield()`中的`list_push_back()`改成`list_insert_ordered()`。

~~~c++
//  list_push_back (&ready_list, &cur->elem);
  list_insert_ordered(&ready_list, &cur->elem, thread_cmp_priority, 0);
~~~
  
<font color="Red">利用有序插入确保ready_list中的线程依据优先级排序。</font>，这样就完成了优先级调度。

根据Pintos要求，仍需要实现基于优先级的抢占式调度。优先级的变动发生在`thread_create()`和`thread_set_priority()`两个函数里，如果前者新建的线程优先级高于当前运行线程，或者当前线程的新优先级不是最高优先级时，重新调度。

在`thread_create()`和`thread_set_priority()`的末尾分别加上：

~~~c++
 if (thread_current ()->priority <= priority)
  {
  thread_yield();
  }
~~~

~~~c++
  if (!list_empty (&ready_list)) {
      struct thread *head = list_entry(list_front(&ready_list), struct thread, elem);
      if (head->priority >= new_priority)
      thread_yield ();
    }
~~~


#### Priority Donation ####


进程调度在上一节有提到是简单的FCFS，thread中虽然有priority，但并无应用。要实现优先级调度的话，只需对ready_list中的线程依据优先级排序。

构造比较函数`thread_cmp_priority`.

~~~c++
bool
thread_cmp_priority (struct list_elem *a, struct list_elem *b, void *aux)
{
    ASSERT (a != NULL && b != NULL);

    struct thread *A = list_entry(a, struct thread, elem);
    struct thread *B = list_entry(b, struct thread, elem);

    return A->priority > B->priority;
}
~~~

然后把`thread_unblock()`，`thread_yield()`中的`list_push_back()`改成`list_insert_ordered()`。

~~~c++
//  list_push_back (&ready_list, &cur->elem);
  list_insert_ordered(&ready_list, &cur->elem, thread_cmp_priority, 0);
~~~

<font color="Red">利用有序插入确保ready_list中的线程依据优先级排序。</font>，这样就完成了优先级调度。

根据Pintos要求，仍需要实现基于优先级的抢占式调度。优先级的变动发生在`thread_create()`和`thread_set_priority()`两个函数里，如果前者新建的线程优先级高于当前运行线程，或者当前线程的新优先级不是最高优先级时，重新调度。

在`thread_create()`和`thread_set_priority()`的末尾分别加上：

~~~c++
 if (thread_current ()->priority <= priority)
  {
  thread_yield();
  }
~~~

~~~c++
  if (!list_empty (&ready_list)) {
      struct thread *head = list_entry(list_front(&ready_list), struct thread, elem);
      if (head->priority >= new_priority)
      thread_yield ();
    }
~~~

#### Priority Donation ####

先了解两个概念：

优先级逆转：线程A、B和C的优先级分别是1,2和3。A持有锁lock，C想要获得锁lock。那么线程C需要等到A释放锁才能获得锁。因为C在等待锁，因此被阻塞。而A和B中优先级更高的B获得了CPU时间。换句话说：优先级更高的C反而在优先级较低的B之后运行。

![Priority Inversion](/images/2015-05-02-Pintos Project1/Inversion.gif ) 

优先级捐赠：处理优先级逆转的一个方法是，当C发现A持有锁lock时候，把A的优先级暂时提升到C的优先级（如果C比A的优先级高），那么A就会在B之前得到CPU，A释放锁后恢复原来优先级。下一个获得CPU时间的将是C，而不是B。

![Priority Donation](/images/2015-05-02-Pintos Project1/Donation1.gif)

![Priority Donation](/images/2015-05-02-Pintos Project1/Donation2.gif)
 
由于文档没有明确具体的要求，所以选择TDD。

以下是关于Priority Donation的测试，其中包含了实现要求，我逐个分析。

~~~c++
FAIL tests/threads/priority-donate-one
FAIL tests/threads/priority-donate-multiple
FAIL tests/threads/priority-donate-multiple2
FAIL tests/threads/priority-donate-nest
FAIL tests/threads/priority-donate-sema
FAIL tests/threads/priority-donate-lower
FAIL tests/threads/priority-sema
FAIL tests/threads/priority-condvar
FAIL tests/threads/priority-donate-chain
~~~

priority-donate-one，<font color="Red">要求一个锁释放后，下一个获得锁的进程按照waiters的优先级决定</font>。

~~~c++
/* The main thread acquires a lock.  Then it creates two
   higher-priority threads that block acquiring the lock, causing
   them to donate their priorities to the main thread.  When the
   main thread releases the lock, the other threads should
   acquire it in priority order.

   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */
~~~
   
锁和信号量数据结构如下：

~~~c++
/* A counting semaphore. */
struct semaphore
  {
    unsigned value;             /* Current value. */
    struct list waiters;        /* List of waiting threads. */
  };
~~~

~~~c++
/* Lock. */
struct lock
{
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
};
~~~

priority-donate-multiple和priority-donate-multiole2,<font color="Red">线程释放锁A和锁B后，放弃对应锁的等待队列中线程带来的捐赠</font>。

~~~c++
/* The main thread acquires locks A and B, then it creates two
   higher-priority threads.  Each of these threads blocks
   acquiring one of the locks and thus donate their priority to
   the main thread.  The main thread releases the locks in turn
   and relinquishes its donated priorities.
   
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */
~~~

~~~c++
/* The main thread acquires locks A and B, then it creates three
   higher-priority threads.  The first two of these threads block
   acquiring one of the locks and thus donate their priority to
   the main thread.  The main thread releases the locks in turn
   and relinquishes its donated priorities, allowing the third thread
   to run.

   In this test, the main thread releases the locks in a different
   order compared to priority-donate-multiple.c.
   
   Written by Godmar Back <gback@cs.vt.edu>. 
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */
~~~
   
priority-donate-nest，<font color="Red">嵌套提升线程优先级。</font>

1. L(A)，L获得锁A
2. M(B) ---> A，M获得锁B，想要A锁，提升A锁持有者L的优先级（如果M比L高），堵塞。
3. H ---> B, H想要B锁，提升B锁拥有者M的优先级（如果H比M高），同时提升拥有M需要的锁的线程的优先级，直到线程没有想要的锁。即H----> M----> 中间线程 ----> L

~~~c++
/* Low-priority main thread L acquires lock A.  Medium-priority
   thread M then acquires lock B then blocks on acquiring lock A.
   High-priority thread H then blocks on acquiring lock B.  Thus,
   thread H donates its priority to M, which in turn donates it
   to thread L.
   
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */
~~~

priority-donate-sema，<font color="Red">基于信号量、锁和优先级造成的抢占式调度</font>。

1. 锁A，信号量S，低优先级线程L， L(A) ---> S，L堵塞
2. 中优先级线程M， M ---> S，M堵塞
3. 高优先级线程H， H ---> A, 提升L优先级，M堵塞。
4. 主线程对S进行V操作，此时S的等待队列中优先级更高的L被唤醒。
5. L抢占主线程，对S进行P操作，释放锁A，H从A的等待队列中被唤醒,L恢复低优先级。
6. H抢占L，获得锁A，对S进行V操作，此时S的等待队列中的M被唤醒，H运行结束。
7. M获得CPU时间，M终止。
8. L获得CPU时间， L终止。

~~~c++
/* Low priority thread L acquires a lock, then blocks downing a
   semaphore.  Medium priority thread M then blocks waiting on
   the same semaphore.  Next, high priority thread H attempts to
   acquire the lock, donating its priority to L.

   Next, the main thread ups the semaphore, waking up L.  L
   releases the lock, which wakes up H.  H "up"s the semaphore,
   waking up M.  H terminates, then M, then L, and finally the
   main thread.

   Written by Godmar Back <gback@cs.vt.edu>. */
~~~

priority-donate-lower，<font color="Red">被捐赠提升优先级的线程无法调用thread_set_priority设定更低的优先级</font>。

~~~c++
/* The main thread acquires a lock.  Then it creates a
   higher-priority thread that blocks acquiring the lock, causing
   it to donate their priorities to the main thread.  The main
   thread attempts to lower its priority, which should not take
   effect until the donation is released. */
~~~

priority-sema，<font color="Red">信号量的等待队列按优先级排序</font>。

~~~c++
/* Tests that the highest-priority thread waiting on a semaphore
   is the first to wake up. */
~~~

priority-condvar<font color="Red">，condition的等待队列也按优先级排序。</font>

~~~c++
/* Tests that cond_signal() wakes up the highest-priority thread
   waiting in cond_wait(). */
~~~

---

开始实现。

`thread`加入以下member。

~~~c++
 /* The priority in the first place */
    int pristine_priority;
    /* The lock that it is waiting for */
    struct lock *blocked_by_lock;
    /* The locks that this thread holds */
    struct list locks_holding_list;
~~~
    
`lock`加入一下member。此处把捐赠的受体对象抽象成lock，而不是线程。线程的优先级由自身的`pristine_priority`和`locks_holding_list`中的锁`lock_priority`共同决定。

~~~c++
    int lock_priority; /* The highest priority of the thread among waiters */
    struct list_elem elem_lock; /* The element for locks_holding_list */
~~~

修改`lock_acquire`。改造成nested donation。

~~~c++
void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  enum intr_level old_level;
  old_level = intr_disable ();

  struct thread* cur = thread_current ();

  if (lock->holder != NULL) {
      cur->blocked_by_lock = lock;
      if (!thread_mlfqs)
      {
          struct lock* l = lock;
          // nested donation
          while (l != NULL && cur->priority > l->lock_priority)
          {
              l->lock_priority = cur->priority;
              thread_update_priority (l->holder);
              l = l->holder->blocked_by_lock;
          }
      }
  }

  sema_down (&lock->semaphore);
  thread_get_lock (lock);
  intr_set_level (old_level);
}
~~~

里面首先调用的`thread_update_priority`定义如下。更新一个线程优先级，当其需要的锁被released或者其拥有的锁被其他线程捐赠。

~~~c++
/*  Update a thread's priority when there is a donation to one of its lock or
    when one of its lock is released
*/
void thread_update_priority (struct thread* t)
{
    enum intr_level old_level = intr_disable ();
    // set priority to the pristine one
    t->priority = t->pristine_priority;
    if (!list_empty (&t->locks_holding_list))
    {
        list_sort (&t->locks_holding_list, lock_cmp_priority, NULL);
        // the highest priority among locks it hold.
        int lock_priority = list_entry (list_front (&t->locks_holding_list), struct lock, elem_lock)->lock_priority;
        // if lock_priority == t->priority, that means no donation
        if (lock_priority > t->priority)
        {
            t->priority = lock_priority;
        }
    }

    if (t->status == THREAD_READY)
    {
        list_sort (&ready_list, thread_cmp_priority, NULL);
    }

    intr_set_level (old_level);
}
~~~

`thread_get_lock()`更新锁和获得锁的线程信息。锁被释放后，捐赠失效，因此重新定义`lock_priority`为`PRI_MIN`。

~~~c++
/* Dealing with a thread getting a lock */
void thread_get_lock (struct lock *lock)
{
    struct thread *cur = thread_current ();

    cur->blocked_by_lock = NULL;
    lock->holder = cur;
    // donation before take no effect any more
    lock->lock_priority = PRI_MIN;
    list_insert_ordered(&cur->locks_holding_list, &lock->elem_lock, lock_cmp_priority, NULL);
}
~~~

`lock_release`重新实现。释放锁后，更新锁原拥有者的优先级。

~~~c++
void
lock_release (struct lock *lock)
{
    ASSERT (lock != NULL);
    ASSERT (lock_held_by_current_thread (lock));
    enum intr_level old_level;
    old_level = intr_disable ();

    // remove the lock from locks_holding_list;
    list_remove (&lock->elem_lock);
    // update the owner's priority after releasing this lock
    thread_update_priority (thread_current ());

    lock->holder = NULL;
    sema_up (&lock->semaphore);

    intr_set_level (old_level);
}
~~~

下面实现优先级调度。改为优先级队列后，**<font color="Red">调用thread_yiele()重新调度</font>**，我忘记了这点导致debug了很久。

~~~c++
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;
  struct thread* other = NULL;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  sema->value++;
  if (!list_empty (&sema->waiters)) {
      list_sort (&sema->waiters, thread_cmp_priority, NULL);
      other = list_entry(list_pop_front(&sema->waiters), struct thread, elem);
      thread_unblock (other);
      if (thread_mlfqs && other->priority > thread_current()->priority) {
          thread_yield ();
      }
  }

  if (!thread_mlfqs && other != NULL &&
      other->priority > thread_current()->priority) {
      thread_yield();
  }
  intr_set_level (old_level);
}
~~~

`sema_down()`只需要改成`list_insert_ordered()`即可。

~~~c++
void
sema_down (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0)
    {
//      list_push_back (&sema->waiters, &thread_current ()->elem);
      list_insert_ordered (&sema->waiters, &thread_current ()->elem, thread_cmp_priority, NULL);
      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}
~~~

`cond_signal()`同理。

~~~c++
void
cond_signal (struct condition *cond, struct lock *lock UNUSED) 
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty (&cond->waiters))
  {
      list_sort (&cond->waiters, cond_cmp_priority, NULL);
    sema_up (&list_entry (list_pop_front (&cond->waiters),
                          struct semaphore_elem, elem)->semaphore);
  }
}
~~~

整个`Priority Donation`用到的比较函数有两个，如下。

~~~c++
bool
lock_cmp_priority (struct list_elem *a, struct list_elem *b, void *aux)
{
    ASSERT (a != NULL && b != NULL);

    struct lock *A = list_entry(a, struct lock, elem_lock);
    struct lock *B = list_entry(b, struct lock, elem_lock);

    return A->lock_priority > B->lock_priority;
}
~~~

~~~c++
bool
cond_cmp_priority (struct list_elem *a, struct list_elem *b, void *aux) {
    ASSERT (a != NULL && b != NULL);

    struct semaphore_elem *A = list_entry(a, struct semaphore_elem, elem);
    struct semaphore_elem *B = list_entry(b, struct semaphore_elem, elem);
    struct thread *A_ = list_entry (list_front(&A->semaphore.waiters), struct thread, elem);
    struct thread *B_ = list_entry (list_front(&B->semaphore.waiters), struct thread, elem);
    return A_->priority > B_->priority;
}
~~~

最终测试结果：

~~~
pass tests/threads/priority-donate-one
pass tests/threads/priority-donate-multiple
pass tests/threads/priority-donate-multiple2
pass tests/threads/priority-donate-nest
pass tests/threads/priority-donate-sema
pass tests/threads/priority-donate-lower
pass tests/threads/priority-fifo
pass tests/threads/priority-preempt
pass tests/threads/priority-sema
pass tests/threads/priority-condvar
pass tests/threads/priority-donate-chain
~~~


---

### Advanced Scheduler ###


要求实现的BSD调度器要求：[http://www.ccs.neu.edu/home/amislove/teaching/cs5600/fall10/pintos/pintos_7.html](http://www.ccs.neu.edu/home/amislove/teaching/cs5600/fall10/pintos/pintos_7.html) 


BSD调度禁止priority donation，并且使用thread_mlfqs这个布尔变量来标志是否使用BSD调度。

整个BSD调度的idea是，**<font color = "Red">获得CPU时间的线程的recent_cpu不断增加，而recent_cpu代表线程获得cpu的时间，而recent_cpu的增加会降低线程的优先级，从而避免了starvation</font>**。

同时增加nice变量，通过调整nice来**人为控制调整线程的优先级**。

至于load_avg，表示可运行的线程数目，参与到recent_cpu的计算当中。

这三个量的计算公式如下：

> The following formulas summarize the calculations required to implement the scheduler. They are not a complete description of scheduler requirements. 

> Every thread has a nice value between -20 and 20 directly under its control. Each thread also has a priority, between 0 (PRI_MIN) through 63 (PRI_MAX), which is recalculated using the following formula every fourth tick:

> priority = PRI_MAX - (recent_cpu / 4) - (nice * 2).

> recent_cpu measures the amount of CPU time a thread has received "recently." On each timer tick, the running thread's recent_cpu is incremented by 1. Once per second, every thread's recent_cpu is updated this way:

> recent_cpu = (2*load_avg)/(2*load_avg + 1) * recent_cpu + nice.

> load_avg estimates the average number of threads ready to run over the past minute. It is initialized to 0 at boot and recalculated once per second as follows:

> load_avg = (59/60)*load_avg + (1/60)*ready_threads.

> where ready_threads is the number of threads that are either running or ready to run at time of update (not including the idle thread).

显然，recent_cpu和load_avg是实数，但pintos不支持实数运算，因此<font color = "Red">使用整数来模拟实数运算，具体实现是通过对实数左移Q位，使其Q位小数位于小数点左边参与整数运算。需要返回结果再右移N位，舍弃小数部分</font>。

具体实现如下：

~~~c++
#ifndef THREADS_FIXED_POINT_H
#define THREADS_FIXED_POINT_H
typedef int32_t fixed_t;
#define P 17
#define Q 14
#define FRACTION 1 << (Q)

/* Fixed-point real arithmetic */
/* Here x and y are fixed-point number, n is an integer */
#define CONVERT_TO_FP(n) (n) * (FRACTION)
#define CONVERT_TO_INT_ZERO(x) (x) / (FRACTION)
#define CONVERT_TO_INT_NEAREST(x) ((x) >= 0 ? ((x) + (FRACTION) / 2)\
                                   / (FRACTION) : ((x) - (FRACTION) / 2)\
                                   / (FRACTION))
#define ADD(x, y) (x) + (y)
#define SUB(x, y) (x) - (y)
#define ADD_INT(x, n) (x) + (n) * (FRACTION)
#define SUB_INT(x, n) (x) - (n) * (FRACTION)
#define MUL(x, y) ((int64_t)(x)) * (y) / (FRACTION)
#define MUL_INT(x, n) (x) * (n)
#define DIV(x, y) ((int64_t)(x)) * (FRACTION) / (y)
#define DIV_INT(x, n) (x) / (n)

#endif
~~~

BSD调度器要求recent_cpu，priority，load_avg的数据以一定时间间隔来更新。

~~~c++
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_wakeup();
  thread_tick ();
  if (thread_mlfqs)
  {
      incremented_recent_cpu ();
      if (0 == ticks % TIMER_FREQ)
      {
          calculate_load_avg ();
          calculate_recent_cpu_foreach ();
      }
      if (0 == ticks % 4)
      {
          calculate_priority_foreach ();
      }
  }
}
~~~

当前运行线程的recent_cpu每tick加1，每隔1秒对所有线程的recent_cpu更新一次。

~~~c++
void
incremented_recent_cpu (void)
{
    struct thread *cur = thread_current ();
    if (cur != idle_thread)
    {
        cur->recent_cpu = ADD_INT (cur->recent_cpu, 1);
    }
}
~~~

~~~c++
void calculate_recent_cpu (struct thread* cur, void *aux)
{
    ASSERT (is_thread (cur));
    if (cur != idle_thread)
    {
        int load = MUL_INT (load_avg, 2);
        fixed_t coef = DIV (load, ADD_INT (load, 1));
        cur->recent_cpu = ADD_INT (MUL (coef, cur->recent_cpu), cur->nice);
    }
}
~~~

~~~c++
void
calculate_recent_cpu_foreach (void)
{
    thread_foreach (calculate_recent_cpu, NULL);
}
~~~

load_avg作为全局变量，同样每1秒更新一次。

~~~c++
void
calculate_load_avg (void)
{
    struct thread *cur = thread_current ();
    int ready_threads = list_size (&ready_list);

    if (cur != idle_thread)
    {
        ++ready_threads;
    }
    load_avg = MUL (DIV_INT (CONVERT_TO_FP (59), 60), load_avg) +
        MUL_INT (DIV_INT (CONVERT_TO_FP (1), 60), ready_threads);
}
~~~

所有线程的优先级每4 ticks更新一次，校正priority使其位于0到63之间。

~~~c++
void
calculate_priority (struct thread* cur, void* aux)
{
    ASSERT (is_thread (cur));
    if (cur != idle_thread)
    {
        cur->priority = PRI_MAX -
            CONVERT_TO_INT_NEAREST (DIV_INT (cur->recent_cpu, 4)) -
            cur->nice * 2;
    }
    if (cur->priority < PRI_MIN)
    {
        cur->priority = PRI_MIN;
    }
    else if (cur->priority > PRI_MAX)
    {
        cur->priority = PRI_MAX;
    }
}
~~~

~~~c++
void
calculate_priority_foreach (void)
{
    thread_foreach (calculate_priority, NULL);
    if (!list_empty (&ready_list))
    {
        list_sort (&ready_list, thread_cmp_priority, NULL);
    }
}
~~~

最后再实现几个已有的查改函数。

`thread_set_nice()`设置new nice后，如果当前线程不具有最高优先级，重新调度。

~~~c++
/* Sets the current thread's nice value to NICE. */
void
thread_set_nice (int nice)
{
    struct thread* cur;
    cur = thread_current ();
    cur->nice = nice;

    calculate_priority (cur, NULL);

  if (cur != idle_thread)
  {
      if (list_entry (list_begin (&ready_list), struct thread, elem)->priority > cur->priority)
      {
          thread_yield ();
      }
  }
}
~~~

`thread_get_nice()`返回当前进程的nice。

~~~c++
/* Returns the current thread's nice value. */
int
thread_get_nice (void) 
{
    return thread_current ()->nice;
}
~~~

`thread_get_load_avg()`返回load_avg。

~~~c++
/* Returns 100 times the system load average. */
int
thread_get_load_avg (void) 
{
    return CONVERT_TO_INT_NEAREST (MUL_INT (load_avg, 100));
}
~~~

`thread_get_recent_cpu()`返回当前进程cpu。

~~~c++
/* Returns 100 times the current thread's recent_cpu value. */
int
thread_get_recent_cpu (void) 
{
    struct thread *cur = thread_current ();
    return CONVERT_TO_INT_NEAREST (MUL_INT (cur->recent_cpu, 100));
}
~~~

最后不要忘了初始化这三个变量。

~~~c++
/* Starts preemptive thread scheduling by enabling interrupts.
   Also creates the idle thread. */
void
thread_start (void) 
{
  /* Create the idle thread. */
  struct semaphore idle_started;
  sema_init (&idle_started, 0);
  thread_create ("idle", PRI_MIN, idle, &idle_started);

  /* Start preemptive thread scheduling. */
  intr_enable ();
  load_avg = CONVERT_TO_FP(0);
  /* Wait for the idle thread to initialize idle_thread. */
  sema_down (&idle_started);
}
~~~

~~~c++
/* Does basic initialization of T as a blocked thread named
   NAME. */
static void
init_thread (struct thread *t, const char *name, int priority)
{
  ASSERT (t != NULL);
  ASSERT (PRI_MIN <= priority && priority <= PRI_MAX);
  ASSERT (name != NULL);

  memset (t, 0, sizeof *t);
  t->wakeup_ticks = 0;
  t->status = THREAD_BLOCKED;
  strlcpy (t->name, name, sizeof t->name);
  t->stack = (uint8_t *) t + PGSIZE;
  t->priority = priority;
  t->magic = THREAD_MAGIC;
  list_push_back (&all_list, &t->allelem);

  t->pristine_priority = priority;
  list_init (&t->locks_holding_list);
  t->blocked_by_lock = NULL;
  t->nice = 0;
  t->recent_cpu = CONVERT_TO_FP (0);
}
~~~

所有测试PASS!!!

![Project1_PASS](/images/2015-05-02-Pintos Project1/Project1_PASS.png) 


---

