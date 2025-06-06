+++
title = "XV6-2020 LAB2-SYSCALL"
date = 2025-06-06

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++
## 前言
首先，我们得先切换为其他分支。
```sh
git fetch
git checkout syscall
make clean
```

## system call tracing (moderate)
我们需要修改的文件有这些：
```tree
|-- user/
|-- |-- user.h
|-- |-- usys.pl
|-- kernel/
|-- |-- proc.h
|-- |-- syscall.c
|-- |-- syscall.h
|-- |-- sysproc.c
```
#### user.h
我们得在这文件里添加这行：
```C
int trace(int);
```
#### usys.pl
这里也是添加这行就可以了，我们做的这俩操作主要是要让用户态能使用这两个系统调用。
```C
entry("trace");
```

#### syscall.h
添加这行，这个叫系统调用号，最好是顺序，不顺序可能没法跑？（你可以试试）
```C
#define SYS_trace 22
```
#### syscall.c
添加这行：
```C
extern uint64 sys_trace(void);
```
然后因为我们需要输出系统调用的名称，所以相应的，我们也得添加字符串数组。
```C
// string for output
static char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace", 
[SYS_sysinfo] "sysinfo",
};
```
然后我们需要稍微修改一下`syscall`函数。
```C
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    
    // output trace
    if(p->mask & (1 << num)) {
      printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
    }
  } else { // error
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
#### sysproc.c
添加一个新的系统调用。
```C
uint64 
sys_trace(void)
{
  // get int from user
  if(argint(0, &myproc()->mask) < 0)
    return -1;

  printf("trace pid: %d\n", myproc()->pid);
  return 0;
}
```
#### proc.c
在`freeproc`函数中顺便清除`mask`。
```C
p->mask = 0;
```
在`fork`函数中也要复制一下`mask`。
```C
np->mask = p->mask;
```

#### 测试
现在你去测试后，应该能拿到和作业需求相同的结果了，如果没有的话，就需要你自己去排查问题了。

### 常见疑问
Q：`extern`是什么意思   
A：`extern`用来声明一个变量或者函数是在其他文件中定义的，这样编译器就知道它的存在了，比如：
```C
extern uint64 sys_trace(void);
```
这就代表`sys_trace`函数在其他地方定义了，我们只是声明一下，让编译器知道它的存在，不然调用时可能就会报错。

Q：为什么除了`user.h`的，其他的`trace`都是接收`void`的？   
A：在用户态，我们调用系统调用时是这样写的：
```C
int trace(int mask);
```
而在内核态，系统调用的是这样：
```C
uint64 sys_trace(void);
```
虽然函数定义中没有参数，但是实际上参数是通过寄存器传递的。
还记得刚刚我们用的`argint`吗？`argint(0, &var)`，这个`0`的意思就是`a0`寄存器，我们要从`a0`寄存器提取出实际的参数，然后复制给`var`。

Q：`usys.pl`文件是什么呀？是汇编吗？不过看着也不像呀。    
A：首先，他不是汇编文件，是`perl`脚本。他的作用是生成`usys.S`，是个汇编文件，我们`make qemu`之后会生成，感兴趣的话可以自己去看看。我复制了一段里面的代码：
```sh
.global trace
trace:
li a7, SYS_trace
ecall
ret
```
这段代码的作用是：
1. 将系统调用号（这里是22）加载到`a7`寄存器。
2. 执行`ecall`指令，在用户态触发系统调用（之后会进入内核态，不过那又是更复杂的事情了...）。
3. 返回到用户态。
