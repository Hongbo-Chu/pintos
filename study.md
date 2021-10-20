<!--
 * @Author: your name
 * @Date: 2021-10-20 20:44:58
 * @LastEditTime: 2021-10-20 23:07:08
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: /pintos/study.md
-->
#pintos学习
##project1
####相关结构体
```c++
//from thread.h
enum thread_status
  {
    THREAD_RUNNING,     /* Running thread. */
    THREAD_READY,       /* Not running but ready to run. */
    THREAD_BLOCKED,     /* Waiting for an event to trigger. */
    THREAD_DYING        /* About to be destroyed. */
  };

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

```



####线程相关
thread_yieldd的实现
```c++
 1 /* Yields the CPU.  The current thread is not put to sleep and
 2    may be scheduled again immediately at the scheduler's whim. */
 3 void
 4 thread_yield (void)
 5 {
 6   struct thread *cur = thread_current ();
 7   enum intr_level old_level;
 8 
 9   ASSERT (!intr_context ());
10 
11   old_level = intr_disable ();
12   if (cur != idle_thread)
13     list_push_back (&ready_list, &cur->elem);
        //放到就绪队列里面去
14   cur->status = THREAD_READY;
15   schedule ();
16   intr_set_level (old_level);
17 }
``` 
`thread_yeild()`第六行调用`thread_current()`
```c++
1 /* Returns the running thread.
 2    This is running_thread() plus a couple of sanity checks.
 3    See the big comment at the top of thread.h for details. */
 4 struct thread *
 5 thread_current (void)
 6 {
 7   struct thread *t = running_thread ();
 8 
 9   /* Make sure T is really a thread.
10      If either of these assertions fire, then your thread may
11      have overflowed its stack.  Each thread has less than 4 kB
12      of stack, so a few big automatic arrays or moderate
13      recursion can cause stack overflow. */
14   ASSERT (is_thread (t));
15   ASSERT (t->status == THREAD_RUNNING);
16 
17   return t;
18 }
```


####中断相关函数：
    bool
     intr_context (void) 
     {
       return in_external_intr;
     }
这里直接返回了是否外中断的标志in_external_intr， 就是说ASSERT断言这个中断不是外中断（IO等， 也称为硬中断）而是操作系统正常线程切换流程里的内中断。$\color{RED}{(也称为软中断）}$
####中断相关tips：
    1 enum intr_level old_level = intr_disable ();
    2 ...
    3 intr_set_level (old_level);
被以上两个语句包裹的内容目的是为了保证这个过程不被中断。