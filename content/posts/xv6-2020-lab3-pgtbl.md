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
不过，做完之后（如果有尽量认真思考的话），你会发现你的水平有了大提升，页表很难，不过也很有趣。     

先提醒一下，如果你还没读`xv6-book`，或是还没看`MIT S6.081`的公开课，我建议你去看一下，页表我学了有差不多1个月吧，很难真的。（也有可能是我太菜了嘤嘤嘤QAQ）      

好，那我们开始吧！

## print a page table (easy)
这个难度是`easy`，事实也确实如此，因为这个是最后的`easy`了，之后就是大boss了。     
我先说一下，这个如果真遇到问题了也别抄我`GITHUB REPO`的，我的比较不一样，输出大概是这样：
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
仔细读过实验要求的肯定能看出这不是作业要求的，而是我自己加的，我先放一下作业要求的输出：
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
检查`PTE`是否为`VALID`，并且没有`READ`、`WRITE`、`EXECUTE`权限。如果是的话说明这是中间页表，如果有那三个权限的话，就是叶子页表了。（指向物理内存的页表）
通过`PTE2PA`获得下一个页表的地址然后递归调用：
```C
if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
```
3. 错误处理：
如果发现是叶子页表，那就要直接内核恐慌了（`kernel panic`），因为一般是开发者乱改页表才会出现这种问题，比起数据遭到乱改，直接死机是更好的选择。
```C
else if(pte & PTE_V){
      panic("freewalk: leaf");
```
你可能不是很理解我在说什么，简单来说就是，如果叶子节点是有效的，那就代表他还是有数据的，我们不应该清空有数据的页表，至少这不是`freewalk`需要做的，这是`proc_freepagetable`做的，感兴趣的可以去`kernel/proc.c`看看。
4. 释放当前页表（包括物理内存里的数据）：
```C
kfree((void*)pagetable); // 这个会执行很多次，有几次取决于你的pagetable总共有多少个 
```
这之后，你的页表们都会被放进`freelist`。
