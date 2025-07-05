+++
title = "XV6-2020 LAB8-LOCK"
date = 2025-07-06

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++

# 前言
这个LAB...第一个和第二个的难度差距有点大，第一个挺简单，第二个难的离谱（；´д｀）ゞ      

这个LAB主要是让我们学习到了锁与性能的关系，降低锁的粒度，以此获得更好的性能。       
学到的东西也不算少，可以说这LAB和我们上一个LAB的关系还挺大的，都挺好玩的，不过不这个难度确实大不少呢。      
简单点来说的话，就是增加程序的并行性，使其的串行性更低，以此获得更好的多核/多线程（这LAB倒是没多线程）性能，虽然现在的CPU单核也很强（我记得苹果的SOC单核超级强），但是多核性能也是可以大大增加使用体验的，相信读者们使用pthread.h时也能很明显的感觉到。

这个LAB还有一部分是为下一个LAB（FILE SYSTEM）做热身，我目前还没怎么读，估计明天就会开始读了，不然我压根理解不了呢我觉得。

# memory allocator (moderate)
第一个要改的是`kalloc.c`，这个相对简单，可以说和上一个LAB的某个小作业差不多。       

## 问题
首先我们先来看看“为什么要改”：
现在`kalloc()`和`kfree()`有一些问题，就是在多核情况下，会有锁竞争问题，倒不是会影响结果，只是会大幅影响性能。       
就是同时只能有一个CPU调用这些函数，我们需要改一下，让这些函数可以同时被多个CPU调用和运行。

## 工作
我们的工作是要让每个CPU都有自己的freelist，并且在自己的freelist为空时，窃取其他CPU的空闲页。        
自然的，锁的数量也得增加，官方有个规定：锁的名字必须以“kmem”开头。      
只要`kalloctest`能完美通过，那就是通过此测试！(｡･∀･)ﾉﾞ

## 提示
官方倒是有给些提示，我总结了一些较为重要的提示：
- 关于**CPU**有多少个的问题，其实用`kernel/param.h`的**NCPU**就好了，不用太在意XV6上有多少CPU
- 让`freerange()`把所有空闲页分配给正在运行的CPU（博主猜测大概率是**CPU 0**，不过这种技术细节先不用太在意）
- 你可以使用`pop_off()`和`push_off()`来控制中断的**开关**
- `cpuid()`会返回当前CPU核心编号
- 可以试试`snprintf()`来对每个锁命名，不过要每个都叫**kmem**也不是不行

## 实现
首先，我们要先升级一下freelist，从本来的全局版升级为多核独立版。        
锁也要对应的增加一下，然后别忘了给锁命名。      
```C
#define LOCKLEN 6 // 锁名长度限制

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  char lock_name[LOCKLEN]; 
  struct run *freelist;
} kmem[NCPU];

void
kinit()
{
  for(int i = 0; i < NCPU; i++){ // 循环锁的初始化
    snprintf(kmem[i].lock_name, LOCKLEN, "kmem %d", i);
    initlock(&kmem[i].lock, kmem[i].lock_name);
  }
  freerange(end, (void*)PHYSTOP);
}
```

这个小作业最难的就是这里的，因为我们要在原版的基础上加上一个窃取机制。      
要注意的是，我们得关闭中断，以确保我们获得的CPUID在执行时是正确的。     
然后我们还得把freelist改为对应的CPU核心编号。
```C
void *
kalloc(void)
{
  struct run *r;

  push_off(); // 关闭中断
  int cpu = cpuid();
  acquire(&kmem[cpu].lock);
  r = kmem[cpu].freelist;
  if(r){
    kmem[cpu].freelist = r->next;
    release(&kmem[cpu].lock);
  }else{
    // 本地没空闲页也得释放，避免死锁
    release(&kmem[cpu].lock);

    // 窃取空闲页（遍历法）
    // 从其他CPU的FREELIST
    for(int nextid = 0; nextid < NCPU; nextid++){
      if(cpu != nextid){
        acquire(&kmem[nextid].lock);
        r = kmem[nextid].freelist;

        if(r){
          kmem[nextid].freelist = r->next;
          release(&kmem[nextid].lock);
          break;
        }

        release(&kmem[nextid].lock);
      }
    }
  }
  pop_off(); // 开启（也有可能不开启，具体去看源代码）中断

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

这个的话相对简单，只要把释放的空闲页插到对应CPU的链表头就好了，这玩意的不严谨性应该给窃取机制的使用次数降低了不少呢（＞人＜；）。
```C
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();
  int cpu = cpuid();
  acquire(&kmem[cpu].lock);
  r->next = kmem[cpu].freelist;
  kmem[cpu].freelist = r;
  release(&kmem[cpu].lock);
  pop_off();
}
```

最后编辑时间：2025/7/6