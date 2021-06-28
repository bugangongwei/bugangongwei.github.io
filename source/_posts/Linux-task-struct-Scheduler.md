---
title: Linux task_struct Scheduler
date: 2021-05-27 15:23:51
tags: [调度器, 源码解析, draveness]
categories: Linux
comments: true
---

Linux 进程/线程调度器的演进, draveness 《系统设计精要》读书笔记

### (1) 初始调度器 · v0.01 ~ v2.4
Linux 中, 无论是线程还是进程, 都由一个统一的结构体 task_struct 表示, 该结构定义在 {% link includes/sched.h  https://github.com/draveness/linux-archive/blob/master/0.01/include/linux/sched.h#L77 %} 中:
{% codeblock kernel/sched.h lang:c highlight:true %}
struct task_struct {
/* these are hardcoded - don't touch */
  long state; /* -1 unrunnable, 0 runnable, >0 stopped */
  long counter;
  long priority;
  long signal;
  fn_ptr sig_restorer;
  fn_ptr sig_fn[32];
/* various fields */
  int exit_code;
  unsigned long end_code,end_data,brk,start_stack;
  long pid,father,pgrp,session,leader;
  unsigned short uid,euid,suid;
  unsigned short gid,egid,sgid;
  long alarm;
  long utime,stime,cutime,cstime,start_time;
  unsigned short used_math;
/* file system info */
  int tty;    /* -1 if no tty, so it must be signed */
  unsigned short umask;
  struct m_inode * pwd;
  struct m_inode * root;
  unsigned long close_on_exec;
  struct file * filp[NR_OPEN];
/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
  struct desc_struct ldt[3];
/* tss for this task */
  struct tss_struct tss;
};
{% endcodeblock %}


调度器的核心逻辑由 {% link kernel/sched.c https://github.com/draveness/linux-archive/blob/master/0.01/kernel/sched.c#L68 %} 中的 schedule() 函数定义;
{% codeblock kernel/sched.c lang:c highlight:true %}
/*
 *  'schedule()' is the scheduler function. This is GOOD CODE! There
 * probably won't be any reason to change this, as it should work well
 * in all circumstances (ie gives IO-bound processes good response etc).
 * The one thing you might take a look at is the signal-handler code here.
 *
 *   NOTE!!  Task 0 is the 'idle' task, which gets called when no other
 * tasks can run. It can not be killed, and it cannot sleep. The 'state'
 * information in task[0] is never used.
 */
void schedule(void)
{
  int i,next,c;
  struct task_struct ** p;
/* check alarm, wake up any interruptible tasks that have got a signal */
  for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
    if (*p) {
      if ((*p)->alarm && (*p)->alarm < jiffies) {
          (*p)->signal |= (1<<(SIGALRM-1));
          (*p)->alarm = 0;
        }

      /* 唤醒获得信号的可中断进程 */
      if ((*p)->signal && (*p)->state==TASK_INTERRUPTIBLE)
        (*p)->state=TASK_RUNNING;
    }
/* this is the scheduler proper: */
  while (1) {
    c = -1;
    next = 0;
    /* NR_TASKS = 64, 该版本的调度器, 任务队列的长度限制为 64 */
    i = NR_TASKS;
    p = &task[NR_TASKS];
    while (--i) {
      if (!*--p)
        continue;
      /* 
       * 从后往前遍历任务队列, 找到 counter 最大的可执行进程
       * counter 表示进程目前可占用的时间片数量
       */
      if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
        c = (*p)->counter, next = i;
    }
    /* 如果 max counter > 0 , 表明队列中存在还没用完时间片的任务, 则跳出循环, 执行 switch_to(next) 切换进程并分配资源 */
    if (c) break;
    /* 如果 max counter == 0, 表明队列中的所有任务都没有可用的时间片, 此时, 对所有任务都分配时间片 */
    for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
      if (*p)
        (*p)->counter = ((*p)->counter >> 1) +
            (*p)->priority;
  }
  switch_to(next);
}
{% endcodeblock %}

Linux 操作系统的计时器会每隔 10ms 触发一次 do_timer 将当前正在运行进程的 counter 减一，当前进程的计数器归零时就会重新触发调度。
{% codeblock kernel/sched.c lang:c highlight:true %}
void do_timer(long cpl)
{
  if (cpl)
    current->utime++;
  else
    current->stime++;
  if ((--current->counter)>0) return;
  current->counter=0;
  if (!cpl) return;
  schedule();
}
{% endcodeblock %}

