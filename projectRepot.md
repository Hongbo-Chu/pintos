<!--
 * @Author: CHB
 * @Date: 2021-10-21 10:52:11
 * @LastEditTime: 2021-10-21 13:23:33
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: /pintos/projectRepot.md
-->
#project1
##general idea of 1.1
原有的`timer_sleep()`一直在调用`thread_yield()`使得线程在就绪和运行间不断切换，于是要将其修改为由运行态到阻塞态。然后通过在每个`time_tick`到来的时候调用`timer_interrupt()`来对于sleep的状态进行检测，`unblock`每个计时结束的线程。
<font color = RED>
系统自带`readylist`和`allList`,所以不用自己实现`blocklist`只需要遍历alllist然后判断状态是不是blocked。
</font>
</br>
<font color=GREEN>
个人的解决方案分为两部分：
·将线程由运行态转为阻塞态
·检测进程，在计时结束将其唤醒</font>
##我的修改：
###一线程的阻塞
####1timer_sleep()
```c++
void timer_sleep(int64_t ticks){
  //新代码将进程放到阻塞态  
  if(ticks<0){
    printf("time error!");
  }
  
  enum intr_level old_level = intr_disable (); //保证中断不被打断
  struct thread *cur_thread = thread_current ();
  cur_thread->block_time = ticks;//阻塞这么长时间
  thread_block();
  intr_set_level (old_level);
}
```
其中给`thread`结构体添加了`block_time`这个属性用于记录sleep的时间。



####2thread_block()

   `timer_sleep()`调用`thread_block()`:
   ```c++
   thread_block (void) 
{
  ASSERT (!intr_context ());
  ASSERT (intr_get_level () == INTR_OFF);

  thread_current ()->status = THREAD_BLOCKED;

  struct thread *t = thread_current();

  list_push_back (&block_list, &t->elem);
  schedule ();

}
   ```
在原`thread_block()`基础上添加了阻塞队列，直接将sleep的线程添加到阻塞队列中。
###二线程的唤醒
####1.timer_interrupt()
```c++
timer_interrupt (struct intr_frame *args UNUSED)//时钟的中断处理函数，在每次
{
  //每个 time_tick调用一次timer_interrupt()

  
  ticks++;
  thread_tick ();//线程时间片计时
  thread_foreach (blocked_thread_check, NULL);
}
```
计时中断函数在每个`time_tick`被调用。我在原有的基础上添加了对于`threads_foreach()`函数的调用，使得在每个time_tick到来的时候,激活`block_thread_check()`函数，对于阻塞的线程进行遍历检查。
####2.thread_foreach_block()
```c++
void
thread_foreach_block (thread_action_func *func, void *aux)
{
  struct list_elem *e;

  ASSERT (intr_get_level () == INTR_OFF);

  for (e = list_begin (&block_list); e != list_end (&block_list);
       e = list_next (e))
    {
      struct thread *t = list_entry (e, struct thread, allelem);
      func (t, aux);
    }
}
```
参考`thread_foreach`函数，我设计了`thread_foreach_block`函数，该函数对于block_list进行遍历（原函数对all_list进行遍历）

<font color=RED>问题:在哪里把线程添加到all_list的</font>
<font color=RED>如果都被添加到all_list了就可以直接对all_list进行遍历</font><br />


####3.block_thread_check()

```c++
void
  blocked_thread_check (struct thread *t, void *aux UNUSED)
  {
    if (t->status == THREAD_BLOCKED && t->block_time > 0)
    {
        t->block_time--;
        if (t->block_time == 0)
        {
           thread_unblock(t);
        }
   }
 }
```
该函数对于阻塞的进程进行检查，要是时间到了就唤醒

####4.thread_unblock
```c++
void
thread_unblock (struct thread *t) 
{
  enum intr_level old_level;

  ASSERT (is_thread (t));

  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  list_push_back (&ready_list, &t->elem);
  t->status = THREAD_READY;
  list_pop_front (&block_list);//阻塞队列出队 

  intr_set_level (old_level);
}

```
在原函数的基础上添加从阻塞队列中出队
