+++
title = "XV6-2020 LAB3-PGTBL"
date = 2025-06-07

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++
## 前言
好，这个就是最难的LAB了（我觉得）。如果说LAB1、2是小学水平，那么LAB3至少都是高中水平了，真的难。      
不过，做完之后，你会发现你的水平有了大提升，页表很难，不过也很有趣。     

先提醒一下，如果你还没读`xv6-book`，或是还没看`MIT S6.081`的公开课，我建议你去看一下，页表我学了有差不多1个月吧，很难真的。（也有可能是我太菜了嘤嘤嘤QAQ）      

好，那我们开始吧！

## print a page table (easy)
这个难度是`easy`，事实也确实如此，因为这个是最后的`easy`了，之后就是大boss了。     
如果你有去看我的`GITHUB REPO`，你可能会觉得我的代码很奇怪，我这里说明一下我的代码的作用。   
我写的那些是用来输出`VA`和`PERM`的，没什么用，我DEBUG是完全没用到`VMPRINT`，那些代码可以不抄，毕竟不是作业要求。    
如果你去修改代码调用我的版本，输出会是这样的：
```C
PAGETABLE:0x0000000087f1c000
|-- VA:0x0000000000000000 PTE:0x0000000021fc6001 PA:0x0000000087f18000 PERM:0x1
|-- |-- VA:0x0000000000000000 PTE:0x0000000021fc5c01 PA:0x0000000087f17000 PERM:0x3073
|-- |-- |-- VA:0x0000000000000000 PTE:0x0000000021fc641f PA:0x0000000087f19000 PERM:0x1055
|-- |-- |-- VA:0x0000000000001000 PTE:0x0000000021fc540f PA:0x0000000087f15000 PERM:0x1039
|-- |-- |-- VA:0x0000000000002000 PTE:0x0000000021fc501f PA:0x0000000087f14000 PERM:0x31
|-- VA:0x0000003fc0000000 PTE:0x0000000021fc6c01 PA:0x0000000087f1b000 PERM:0x3073
|-- |-- VA:0x0000003fffe00000 PTE:0x0000000021fc6801 PA:0x0000000087f1a000 PERM:0x2049
|-- |-- |-- VA:0x0000003fffffe000 PTE:0x0000000021fed807 PA:0x0000000087fb6000 PERM:0x2055
|-- |-- |-- VA:0x0000003ffffff000 PTE:0x0000000020001c0b PA:0x0000000080007000 PERM:0x3083
```
我放一下作业要求的输出：
```C
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

好，废话很多了，我们来开始做第一个实验吧。       
首先，我们先来看一下`freewalk`函数：
```C
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```
他的作用就是：
1. 遍历512个PTE：
```C
for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
```
2. 处理中间页表：
检查`PTE`是否为`VALID`，并且没有`READ`、`WRITE`、`EXECUTE`权限。如果是的话说明这是中间页表，如果有那三个权限中的任意一个的话，就是叶子节点了。（指向物理内存的页表）
通过`PTE2PA`获得下一个页表的地址然后递归调用：
```C
if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
```
3. 错误处理：
如果发现是叶子页表，那就要直接内核恐慌了（`kernel panic`），因为一般是开发者自行更改页表才会出现这种问题，比起数据遭到预期以为的修改，直接死机是更好的选择。
```C
else if(pte & PTE_V){
      panic("freewalk: leaf");
```
我说的可能有点绕，简单来说就是，如果叶子节点是有效的，那就代表它还是有数据的，我们不应该清空有数据的页表。    
至少这不是`freewalk`需要做的，这是`uvmunmap`做的，感兴趣的可以去`kernel/proc.c`和`kernel/vm.c`看看。
4. 释放当前页表：
```C
kfree((void*)pagetable); // 这个可以执行很多次，有几次取决于你的pagetable总共有多少个 
```
这之后，你的页表们都会被放进`freelist`。

好，可以开始做实验了。    
首先，去打开`kernel/vm.c`。   
然后，实验代码就是这个：
```C
void
vmprint_ori(pagetable_t pagetable, int level)
{
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if (!(pte & PTE_V)) continue;

    for (int i = 3; i > level; i--) {
      printf("..");
      if(i-1 > level){
        printf(" ");
      }
    }
    printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      
    if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
      uint64 child = PTE2PA(pte);
      vmprint_ori((pagetable_t)child, level - 1);
    }
  }
}

void 
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprint_ori(pagetable, 2);
}
```
然后我们打开`kernel/exec.c`。   
实验代码如下：
```C
// Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);

  // 添加这个
  if(p->pid == 1)
    vmprint(p->pagetable);
```
然后去跑跑TEST，应该就可以通过了，那个回答问题的测试你自己`touch`一个然后写答案就可以了。

## a kernel page table per process (hard)
好，这个就是卡我最久的LAB了，是真的难。（2个月左右）

好！我们先来看看这个LAB是要做什么吧！   
首先，XV6的页表现在是怎么样的呢？   

XV6的页表是这样的：
每个用户进程，在用户态运行时，都使用自己的用户态页表。    
但是，一旦进入内核态，就要切换到内核页表，现在这个内核页表是全局的。    

这个实验要我们做的就是，让每个用户进程都有自己的内核页表。    
嘛不过，分两步，这个LAB和下一个关系很大，不可以先去做下一个LAB！    
这个LAB要做的是让XV6从全局内核页表进化为全局+多个副本，并且效果和现在一样。（就是不报错，所有功能正常）   
下一个LAB才是让完善这种进程页表。   

这个LAB会改变的：   
- ❎ 内核页表有各自用户页表的映射
- ✅ 每个用户进程在内核态将会使用自己的内核页表

好，开始吧！    
首先我们要先让我们的进程有一个新的字段。    
和`pagetable`差不多，不过是内核的，所以就叫`kpagetable`吧！   
在`kernel/proc.h`加上新字段：
```C
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  
  pagetable_t kpagetable;      // 添加这个，以后我们的内核页表就是这个了
};
```
然后我们要模仿`kvminit`，这样就能给进程内核页表分东西了。
```C
pagetable_t
kvmmake(int global) 
// 这个参数主要是用来区分全局和副本的
// 副本放CLINT会报错，具体怎么修复我也不知道，目前已知的就是不分配
{
  pagetable_t kpagetable = (pagetable_t) kalloc();
  memset(kpagetable, 0, PGSIZE);

  // uart registers
  mappages(kpagetable, UART0, PGSIZE, UART0, PTE_R | PTE_W);

  // virtio mmio disk interface
  mappages(kpagetable, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W);

  if(global == 1)
  // CLINT
  mappages(kpagetable, CLINT, 0x10000, CLINT, PTE_R | PTE_W);

  // PLIC
  mappages(kpagetable, PLIC, 0x400000, PLIC, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  mappages(kpagetable, KERNBASE, (uint64)etext-KERNBASE, KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  mappages(kpagetable, (uint64)etext, PHYSTOP-(uint64)etext, (uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  mappages(kpagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X);

  return kpagetable;
}

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
  kernel_pagetable = kvmmake(1);
}
```
接下来，我们要把这个副本放进进程里，在`kernel/proc.c`的`allocproc`函数添加以下代码：    
```C
 // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // 添加这个
  p->kpagetable = kvmmake(0); // 正如刚刚所说的，这是副本，所以用参数0
  if(p->kpagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
```
