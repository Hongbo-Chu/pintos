<!--
 * @Author: your name
 * @Date: 2021-10-26 23:02:28
 * @LastEditTime: 2021-10-27 13:55:46
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: /pintos/quesions.md
-->
#questions
###1. thread_init() &init_thread()区别
   1. `inti_thread`被`thread_init()`调用
   2. `thread_init`会在整体init.c中被调用
###2. elem&allelem区别
   1. allelem存储的是allLIst的上下文
   2. elem存储的是readylist的上下文
###3. thread_yield和thread_unblock区别
```c++
thread_yield (void) 
//thread_yield其实就是把当前线程扔到就绪队列里， 然后重新schedule
{
  struct thread *cur = thread_current (); //  当前线程
  enum intr_level old_level;
  
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread) 
    //list_push_back (&ready_list, &cur->elem);
    list_insert_ordered (&ready_list, &cur->elem, (list_less_func *) &thread_cmp_priority, NULL);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
```
```c++
void
thread_unblock (struct thread *t) //放到readylist中
{
  enum intr_level old_level;

  ASSERT (is_thread (t));

  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  // list_push_back (&ready_list, &t->elem);
  list_insert_ordered (&ready_list, &t->elem, (list_less_func *) &thread_cmp_priority, NULL);
  t->status = THREAD_READY;
  intr_set_level (old_level);
}
```
区别：unblock是针对通过参数传入的指定进程t，而yield这是将当前正在运行的线程放入readylist中
###4.  pintos的运行方式（抢占or非抢占）线程部分的总体运行代码
pintos的运行机制为时间片抢占式，每过固定额时间片就被抢占
<br/>
在每个`time_tick`到来的时候调用`timer_interrupt()`(Timer interrupt handler)来对于sleep的状态进行检测,而`timer_interrupt()`会调用`thread_tick (void) `


<br/>
在`thread.h`中
```c++
#define TIME_SLICE 4            /* # of timer ticks to give each thread. */
static unsigned thread_ticks;   /*
```
定义了：4个timer_tick为一个time_slice
<br/>
每过4个tick`thread_tick()`就调用一次`intr_yield_on_return()`
```c++
thread_tick (void) 
{
  struct thread *t = thread_current ();

  /* Update statistics. */
  if (t == idle_thread)
    idle_ticks++;
#ifdef USERPROG
  else if (t->pagedir != NULL)
    user_ticks++;
#endif
  else
    kernel_ticks++;

  /* Enforce preemption. */
  if (++thread_ticks >= TIME_SLICE)
    intr_yield_on_return ();
}
```



###5.  static_ticks&schedule_ticks
###6.  typedef void intr_handler_func (struct intr_frame *);


thread:
intr_handler 
thread_tick

timer:
timer_interrupt 