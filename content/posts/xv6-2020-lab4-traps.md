+++
title = "XV6-2020 LAB4-TRAPS"
date = 2025-06-15

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++

## risc-v assembly (easy)
这个LAB没有多难，看汇编而已，如果有一些经验的的话几分钟就能大致理解了。       
首先，我们先生成`call.asm`，然后我们就能依次看问题了！        
基本只要看这部分就行了：        
```sh
000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7c050513          	addi	a0,a0,1984 # 7e8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	610080e7          	jalr	1552(ra) # 640 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	27e080e7          	jalr	638(ra) # 2b8 <exit>
```

### 第一个问题：
Which registers contain arguments to functions? For example, which register holds 13 in `main`'s call to `printf`? 

中文翻译：    
哪些寄存器用于传递函数参数？比如，`main`调用`printf`时，数字13存在于哪个寄存器中？

解答：    
明显，很明显，我们先找到`13`在哪，在`a2`那里，所以答案就是`a2`。
```sh
li a2, 13 # 了解x86的读者可以暂时理解为mov a2, 13
```

### 第二个问题：
Where is the call to function `f` in the assembly code for `main`? Where is the call to `g`? (Hint: the compiler may inline functions.) 

中文翻译：    
在这个汇编的`main`函数，`f`和`g`在哪被调用？

解答：
```sh
26:	45b1    li a1, 12
```
在地址为26的地方调用了（f(8)+1）的常数，编译器把这个数字优化了。

### 第三个问题：
At what address is the function `printf` located? 

中文翻译：    
`printf`函数的地址位于？

解答：    
```sh
34:	610080e7   jalr	1552(ra) # 640 <printf>
```
注释写了，是`0x640`。

### 第四个问题：
What value is in the register `ra` just after the `jalr` to `printf` in `main`? 

中文翻译：    
在`main`函数中，执行了`jalr`跳转到`printf`之后，`ra`寄存器的值是多少？

解答：    
```sh
34:	610080e7          	jalr 1552(ra) # 640 <printf>
exit(0);
38:	4501                li a0,0
```
`ra`保存的是下一条指令，所以值是`38`，就是`34`+`4`。

### 第五个问题：
Run the following code.
```C
  unsigned int i = 0x00646c72;
  printf("H%x Wo%s", 57616, &i); 
```
What is the output? 

The output depends on that fact that the `RISC-V` is `little-endian`. If the `RISC-V` were instead `big-endian` what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?

中文翻译：    
跑一下这个代码。
```C
  unsigned int i = 0x00646c72;
  printf("H%x Wo%s", 57616, &i); 
```
输出会是什么？    
有那样的输出，是因为`RISC-V`是`小端`。    
那如果`RISC-V`是`大端`呢？你应该怎么设置`i`才能让其拥有相同的输出？   
你是否有必要修改`57616`为其他的值？   

解惑：    
我们先了解什么是小端，什么是大端， 
以这个地址为例，`0x00646c72`，小端和大端的存储方式有些不同。
- 小端：72 6c 64 00   
- 大端：00 64 6c 72   

解答：    
首先，输出会是`He110 World`。   
如果是大端的话，只要设置为`0x726c6400`就可以了。    
没必要修改那个常量了，小端大端都一样的那个。

### 第六个问题：
In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
```C
  printf("x=%d y=%d", 3);
```

中文翻译：    
在下面这个代码，输出了`y=`后，会输出什么数字？不用精确到某个特定数字。
```C
  printf("x=%d y=%d", 3);
```

解答：    
几乎随机的数字，是`a2`寄存器当前的值，很难预测。    
至少我们是很难调整的，除非我们自己改GCC，或者自己写一个奇怪的编译器。

最后编辑时间：2025/6/16