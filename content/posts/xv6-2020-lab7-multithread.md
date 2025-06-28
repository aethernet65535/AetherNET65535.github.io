+++
title = "XV6-2020 LAB7-MULTITHREADING"
date = 2025-06-28

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++

# 前言
这个LAB有些特别呢，我们后面两个小作业都是在C标准库做的，就是说，我们不是调用XV6的头文件了，而是像我们刚开始学C那时候一样，用标准库！(ﾉ>ω<)ﾉ       

这个LAB有几个要改的文件，分别是：`user/uthread.c` `user/uthread_switch.S`、`notxv6/ph.c`和`notxv6/barrier.c`。        

# uthread: switching between threads (moderate)
第一个小作业呢，是要我们在一个进程内，模拟有好多小进程在跑。(づ′▽`)づ       
这些小进程是有名字的，叫做进程（threads）。       

读者们可能看过内核目录里的`swtch()` `struct context`之类的代码，LAB-3（pgtbl）那时候卡着的话，应该是肯定看过这些代码的吧？如果看过的话，这个小作业就很简单了！*ଘ(੭*ˊᵕˋ)੭* ੈ✩‧₊˚     

## 上下文的定义
首先我们要复制`kernel/proc.h`的`struct context`结构体！     

然后我们要粘贴到`user/uthread.c`：
```C
struct context {
  uint64 ra; // 这两个寄存器很重要哦
  uint64 sp; // 先记住，要用到的

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

之后在我们的结构体里添加一下：
```C
struct thread {
  char              stack[STACK_SIZE];  /* the thread's stack */
  int               state;              /* FREE, RUNNING, RUNNABLE */
  struct context    context;            /* 加上这个，空格要不要对齐都是可以的 */
};
```

## 切换上下文

然后我们现在就去`kernel/swtch.S`复制一下一段代码，然后无脑粘贴进`user/uthread_switch.S`就行了：
```C
thread_switch:
        /* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret    /* return to ra */
```

然后现在，我们把那个**extern**稍微修改一下：
```C
extern void thread_switch(struct context*, struct context*);
```

## 小修小补
现在呢，我们得稍微修一些函数，因为XV6故意留空了。    
不会很难的，就一个函数有点难度，读者们看看就懂了！   

### 调度
首先是上下文切换：
```C
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch(&t->context, &next_thread->context);
  } else
    next_thread = 0;
}
```
这里调用`thread_switch()`是什么意思呢？    
其实就是把旧线程保存起来，然后加载新线程而已！

为什么这个不难呢？   
因为它的提示给的太明显了，都告诉我们要调用什么了！   

### 新线程创建
这个就稍微有点难了，但是我刚刚给了读者们提示！   
就是**ra**和**sp**寄存器，这就是我们为数不多需要修改的东西了！   

```C
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.sp = (uint64)t->stack + STACK_SIZE;
  t->context.ra = (uint64)func;
}
```
为什么要改**sp**呢？   
因为是新线程，它有独立的栈，然后栈不是高地址的吗？但是！在C里，全局/静态结构体的所有字段的初始值都是0，这是超级低地址，所以我们要当然就要把**sp**指向栈低啦，就要+STACK_SIZE，这样就可以指向栈低了！    

那为什么又要修改**ra**呢？这玩意不返回地址嘛？    
确实，**ra**是返回地址，但是在这里有妙用！    
读者想想，当一个代码没有其他代码，只有一个**return**的话，会发生什么？    
呃呃...可能会报错，不过我们不管那个！在这里它的作用是直接前往那里！   
也就是说，线程一创建，制定**ra**后，它就会跑去`func()`那里了，它就能跑我们给他的函数，很厉害对吧？ヾ(´︶`*)ﾉ♬   

最后编辑时间：2025/6/28
