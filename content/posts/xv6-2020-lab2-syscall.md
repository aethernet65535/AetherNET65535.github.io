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
我们得在这文件里添加这行。
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
添加这行。
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
Q：`extern`是什么意思？        
A：`extern`修饰符在这里的意思是，声明一个变量的存在，但不定义它。它会告诉编译器“这个变量在其他地方定义了，你现在看到的只是一个声明”

Q：为什么除了`user.h`的，其他的`trace`都是接收`void`的？
A：我不知道呀，不过我们在`sysproc.c`那里实现的确实有接收`int`参数，我想其他的都是`void`可能是什么规定吧，而且系统上功能确实可以正常使用。

Q：`usys.pl`文件是什么呀？是汇编吗？不过看着也不像呀。
A：似乎是这样的，我们在`user.h`提供给用户API对吧？然后`usys.pl`就会将`trace`的系统调用号放到`a7`寄存器，之后我们就能用让用户态和内核态连接了，大概（我也不知道呀）

Q：那么...我们是怎么从用户态转到内核态的呢？
A：这个我也不知道，我想是这样。CPU把我们送过去`syscall`函数那里，然后就跳到`sysproc.c`执行我们的系统调用，应该是这样吧？
