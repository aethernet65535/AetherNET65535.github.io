+++
title = "XV6-2020 LAB5-LAZY"
date = 2025-06-21

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++

## 前言
这个LAB的平均难度和LAB4（TRAPS）差不多，就是稍微有点难。不过我们可以通过看Hints来有个大致上的思路。     

### 什么是 **Lazy Allocation惰性分配）** ？
Lazy Allocation的功能和它的名字有些像，就是不到最后他都不做，拖延症老严重了呢。(⁠•⁠ ⁠▽⁠ ⁠•⁠;⁠)      

但是呢，在内存分配，有一点点的拖延症并不是坏事，只要做好工作就是好事，不要让程序和用户察觉到就可以了。      

重点来了，Lazy和XV6现在的内存分配方案有什么不同？（我们先不说释放内存）     
- **XV6：** Foo进程想要内存，所以他调用`sbrk`，而`sbrk`则会调用`growproc`，它会给该进程的`sz`字段增值，然后直接给Foo 进程`kalloc()`，就是直接把页表借给他用。
- **Lazy：** Foo进程调用`sbrk`，`sbrk`就会给进程`sz`增值，但是不给他`kalloc()`，因为他知道进程还没用到，所以`sbrk`就表示不急。

之后，当Foo进程开始运作时，他就会告诉计算机说“我要对`0x65535`写入信息”，然后计算机就开始计算“这个进程中的`0x65535`真正指向哪里呢？”，但是计算机会发现这个虚拟地址是无效的，并把这个无效虚拟地址写入`stval`寄存器，然后就没然后了，不过XV6倒是会处理，XV6的处理方案是：报错      

## eliminate allocation from sbrk() (easy)
这个确实没骗人，就是`easy`难度。我们只要搞破坏就行了，不过因为之后的作业也会用到这个系统调用，所以我们得精心的“搞破坏” ٩(｡・ω・｡)و

首先呢，XV6要我们做什么？       
1. 删除`sbrk`对`growproc`的调用     
2. 只修改进程的`sz`值（sz + n）     

### 原代码
首先我们先看原代码，然后才开始思考我们要做的事情。
`kernel/sysproc.c`：
```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(growproc(n) < 0)
    return -1;
  return addr;
}
```
这代码实现了以下操作：
1. 获取用户想要被分配/释放的内存字节量
2. 获取当前进程的`sz`值
3. 调用`growproc`，视情况分配/释放内存
4. 返回当前进程的`sz`值，即新内存的起始地址

#### 嗯？为什么`sz`是新内存的起始地址呢？它不是旧内存的结尾地址吗？
众所周知，编程有两大难题，其中一个是差一错误：
好，首先我们先忽略内存对齐。

如果我们有一个进程，他的`sz = 1`，那么它能用什么虚拟地址呢？        
你可能会觉得是`0x01`，这不是正确答案。      
仔细想一下，如果`sz = 0`呢？它应该可以使用`0x00`吗？        
肯定是不行的，`0`应该是不能使用任何内存，对吧？     

所以`sz` == `可用地址的结束地址 + 1`。      
ヾ(^▽^*)))

### 删除`growproc`      
现在我们先把对`growproc`给删了，现在代码变成这样：      
```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  
  return addr;
}
```

### 修改进程的`sz`值        
然后别忘了这步，这步很重要呀！！        
```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;

  struct proc *p = myproc();
  addr = p->sz;

  p->sz += n;
  
  return addr;
}
```

### 释放内存（提前准备）
这个我不知道要怎么解释，反正就是，你之后肯定会出BUG的，如果你这样写的话，所以如果你怕你以后出BUG的话，就可以先抄成这样，如果想要自己解决BUG的话，可以跳过。     

```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;

  struct proc *p = myproc();
  addr = p->sz;

  if(n < 0){
    uvmdealloc(p->pagetable, p->sz, p->sz+n);
  }
  p->sz += n;

  return addr;
}
```

### 运行结果
```C
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x0000000000001258 stval=0x0000000000004008
va=0x0000000000004000 pte=0x0000000000000000
panic: uvmunmap: not mapped
```
运行结果可能会是这样，不用太过担心，这是应该的，我们不必太过在意这个报错，直接开始下一个作业就行了！        

## lazy allocation (moderate)
这个小作业很好玩的，而且也很简单，就是跟着hints走而已。     

### 要求
首先，我们要做什么呢？      
刚刚报错了对吧？所以我们现在要做的就是，不让`sbrk`直接或间接调用`growproc`，但是要让他不会报错，并且可以正确的执行`echo hi`       

#### 提示
- 此类错误发生时，会修改`scause`寄存器，如果是`13`那就是load page fault，`15`就是store/AMO page fault       
-  `stval`会记录计算机试图访问什么虚拟地址时失效了      
- 你可以参考`uvmalloc`的代码，核心是`kalloc()`+`mappages()`     
- 使用`PGROUNDDOWN`宏对齐到该虚拟地址4KB内的起始地址，简称向下取整      
- `uvmunmap`可能会导致kernel panic（你可以直接跳过某些检查）        

### 缺页的正确处理方法
现在XV6的缺页处理是报错，所以呢，我们得改成，缺页了就尝试给它分页，当然，它得是合法的才行。   
我们得保证只有在堆（HEAP）范围内的内存使用LAZY ALLOCATION，而不是整个XV6都用，毕竟`sbrk`本来就是只给堆用的。    

我们先看看，当触发PAGE FAULT时，XV6会在哪里报错？
我们看看`kernel/trap.c`的`usertrap`函数：   
```C
} else if((which_dev = devintr()) != 0){  // 2 是计时器中断，1 是硬件中断，0 是未识别（就是没中断，不是软件中断）
    // ok
  } else { // 就是我们看到的的那个报错
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

现在，我们来做新的处理方法，但是报错可以保留，万一真的错了呢。
```C
} else if((which_dev = devintr()) != 0){
    // ok

  // 如果scause存储的13/15
  } else if (r_scause() == 13 || r_scause() == 15){
    uint64 stval = r_stval(); // 读取无效虚拟地址
    lazyalloc(stval, p); // 分配内存！
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

### 分配内存
这个你可以选择不封装，当然也可以选择封装，我的建议当然是封装起来，毕竟这函数可以让不少地方调用。    

```C
int
lazyalloc(uint64 stval, struct proc *p)
{
  uint64 va = PGROUNDDOWN(stval);
  if(stval >= p->sz || stval < PGROUNDDOWN(p->trapframe->sp))
    goto err;

  return _lazyalloc_internal(va, p, 0);

  err:
  p->killed = 1;
  return -1;
}
```

```C
// mode 0: trap
// mode 1: read/write
static int
_lazyalloc_internal(uint64 va, struct proc *p, int mode)
{
  char *mem = kalloc();
  if(mem == 0)
    goto err;

  memset(mem, 0, PGSIZE);
  if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_U) != 0){
    kfree(mem);
    goto err;
  }

  return 0;

  err:
  if(mode == 0)
    p->killed = 1;
  return -1;
}
```

这个代码的作用是：      
对**无效虚拟地址**向下取整，然后检查是否大于进程可以拥有的虚拟地址，再检查是否小于堆的底部    
（堆的底部是栈，栈只有一页，PGROUNDDOWN(sp)可以很好的计算出堆的底部）

注：XV6的栈是低地址，堆是高地址   
`kernel/memlayout.h`
```C
// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
```

#### 为什么我要用两个函数呢？说简单点就是             
因为下一个小作业我们又要做一个新的函数了，功能上虽然有些重叠，但是我不想让他们的名字相同。      
我一开始是有合并的，但是我之后想了想，还是拆分成这样了。        
毕竟一个是处理陷阱的，一个是处理读写的，我想让调用方更容易理解代码，并且不让代码很乱或很重复。

### 跳过部分检查
因为现在不是以前的传统分配了，所以有一些检查会导致我们失败。    
因为以前的检查是默认大部分用户的虚拟地址是可以直接映射到物理地址的。    
所以我们得跳过一些检查了，这是较为简单的方法。    
`vm.c/uvmunmap`
```C
for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
  if((pte = walk(pagetable, a, 0)) == 0) continue;
  //  panic("uvmunmap: walk");
  if((*pte & PTE_V) == 0) continue;
  //  panic("uvmunmap: not mapped");
```
`vm.c/uvmcopy`
```C
for(i = 0; i < sz; i += PGSIZE){
  if((pte = walk(old, i, 0)) == 0) continue;
  //  panic("uvmcopy: pte should exist");
  if((*pte & PTE_V) == 0) continue;
  //  panic("uvmcopy: page not present");
```

现在你输入`echo hi`时，应该会看到系统输出`hi`。

最后编辑时间：2025/6/22
