+++
title = "XV6-2020 LAB1-UTIL"
date = 2025-06-04

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++
事先声明，我写的博客，操作基本都是在ARCH LINUX上实现的。

这是XV6-2020的第一个LAB，主要是初步认识RISC-V版的XV6，然后为其实现一些用户态的小工具。

这是我的GitHub链接，如果有关于这个项目的问题，可以试试查看[这里](https://github.com/aethernet65535/DOCKER-XV6_2020)

## boot xv6 (easy)
首先，先下载`git`，再`clone`下来`xv6-labs-2020`的文件。
```sh
sudo pacman -S git

git clone git://g.csail.mit.edu/xv6/labs-2020
cd xv6-labs-2020
git checkout util
make clean && make qemu
```

<br>

> 如果跑不了的话，可能是因为你的QEMU太新了，也有可能是其它问题。你可以试试给QEMU降级，或者去学习使用PODMAN或DOCKER（并不难）。

## sleep (easy)

在`user`目录里，创建名为`sleep.c`的文件。
```tree
xv6-labs-2020/
|-- user/
|-- |-- sleep.c
```

调用sleep系统调用来实现这个功能。
```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main (int argc, char *argv[])
{
    // if the argument is not 2 (sleep <ticks>)
    if (argc != 2)
    {
        fprintf (2, "usage: sleep <ticks>\n");
        exit(1);
    }
    
    // change ascii to integer
    int ticks = atoi(argv[1]);

    // turn back time: disabled
    if (ticks < 0)
    {
        fprintf (2, "ticks must be a positive integer\n");
        exit(1);
    }

    sleep(ticks);

    exit(0);
}
```

在`Makefile`的`UPROGS`处添加`$U/_sleep\`
```Makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep\ # 加上这个
```

## pingpong (easy)

和上一个一样，先在`user`目录里创建`pingpong.c`文件。

```tree
xv6-labs-2020/
|-- user/
|-- |-- sleep.c
|-- |-- pingpong.c
```

<br>

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define READ 0
#define WRITE 1

int main (int argc, char *argv[])
{
    if (argc != 1)
    {
        fprintf(2, "usage: pingpong\n");
        exit(1);
    }

    int pid;

    // one-way pipes for communication
    // p2c: parent to child
    // c2p: child to parent
    int p2c[2], c2p[2]; 
    char signal = 0;
    
    // create pipes
    pipe(p2c); 
    pipe(c2p);

    pid = fork();

    if (pid > 0) // parent process
    {
        // parent process just needs to write[p2c] and read[c2p]
        close(p2c[READ]);
        close(c2p[WRITE]);

        // tee off: send a byte signal to child
        write(p2c[WRITE], &signal, 1);
        close(p2c[WRITE]);

        // waiting for catch: blocking until child sends a byte signal
        read(c2p[READ], &signal, 1);
        close(c2p[READ]);

        printf("%d: Received Pong\n", getpid());
        
        exit(0);
    }
    else if (pid == 0) // child process
    {
        // child process just needs to read[p2c] and write[c2p]
        close(c2p[READ]);
        close(p2c[WRITE]);

        // waiting for catch: blocking until parent sends a byte signal
        read(p2c[READ], &signal, 1);
        close(p2c[READ]);

        printf("%d: Received Ping\n", getpid());
        
        // tee off: send a byte signal to parent
        write(c2p[WRITE], &signal, 1);
        close(c2p[WRITE]);

        exit(0);
    }
    else
    {
        fprintf(2, "fork error\n");
        exit(1);
    }
}
```
上面的代码你可能看不懂，让我解释解释。
<br>
首先，我们当前的进程`fork`出了一个子进程后，这两个进程是并行的，就是他们是同时跑的。
<br>
但是我们希望这个代码是有顺序的，所以我们需要让其中一个不能跑。
<br>
我们在这里利用的是管道的堵塞机制，即“如果读取不到，就不执行下一个代码”。

用乒乓球举例（我没打过，也不了解乒乓球）
<br>
乒乓球的桌子，上面有两个区域，一个是P区域，一个是C区域。
<br>
父亲站在P那里，他只可以把球打过去C那里，或者抓住飞到P区域的球。
<br>
一样的，儿子只能站在C那里，他只能把球打去P区域，或者抓住飞到C区域的球。

当父亲还没有把球打给儿子时，儿子就不能抓住球，因为球没有飞向他。

### 常见疑问

Q: 为什么需要两个管道？     
A: 管道是单向的，要实现双向通信必须用两个管道。

Q: `signal`的作用是什么？   
A: 实际不需要传输有效数据，只需通过管道的**存在性**触发同步。

Q: 不关闭未使用的描述符会怎样？  
A: 可能导致——子进程的`read`无法收到EOF（因为父进程的写入端未关闭）

## primes(moderate)/(hard)

就看代码的话，不算难，只要记住一句话即可：`一个2到35的管道，我们要用的进程数量大约是这里所有素数，一个进程输出一个素数。`

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define READ 0
#define WRITE 1

__attribute__((noreturn)) // to avoid warning about not returning
void sieve_algo (int left[2], int depth)
{
    close(left[WRITE]);

    int prime, temp, pid, right[2];

    // read a prime number from the pipe
    if (read(left[READ], &prime, sizeof(int)) == 0)
    {
        close(left[READ]);
        exit(0);
    }
    
    // if the depth is too deep, we have a stack overflow
    if (depth > 15)
    {
        fprintf(2, "stackoverflow\n");
        close(right[READ]);
        close(right[WRITE]);
        close(left[READ]);
        exit(1);
    }

    printf("prime: %d\n", prime);
    pipe(right);

    pid = fork();
    if (pid > 0) // parent process
    {
        close(right[READ]);

        while(read(left[READ], &temp, sizeof(int)))
        {
            if (temp % prime != 0)
            {
                if (write(right[WRITE], &temp, sizeof(int)) != sizeof(int))
                {
                    fprintf(2, "write error\n");
                }
            }
        }
        close(right[WRITE]);
        wait(0);
        exit(0);
    }
    else if (pid == 0) // child process
    {
        close(left[READ]);
        close(right[WRITE]);
        sieve_algo(right, depth + 1);
        exit(0);
    }
    else // fork failed
    {
        fprintf(2, "fork error\n");
        close(right[READ]);
        close(right[WRITE]);
        close(left[READ]);
        exit(1);
    }
}

int main(int argc, char* argv[])
{
    int pid, p[2];
    pipe(p);

    pid = fork();
    if (pid > 0) // parent process
    {
        close(p[READ]);
        
        // write 2 to 35 to the pipe
        for (int i = 2; i <= 35; i++)
        {
            write(p[WRITE], &i, sizeof(int));
        }
        
        // p finish working
        close(p[WRITE]);
        wait(0); 
        exit(0); 
    }
    else if (pid == 0) // child process
    {
        sieve_algo(p, 1);
        exit(0);
    }
    else // fork failed
    {
        fprintf(2, "fork error\n");
        exit(1);
    }
}
```

### 常见疑问
Q：为什么`sieve_algo`函数里，子进程不等待父进程就直接进入递归？     
A：因为在`sieve_algo`的某段代码，会让子进程被堵塞，但是此时父进程还没有关闭输入端，所以子进程只会堵塞，不会报错。       

```C
if (read(left[READ], &prime, sizeof(int)) == 0) // 这里会堵塞
```

## find (moderate) 
说实话，这个其实不难，基本上CV再改改就行了。        
但是，如果你不知道怎么做的话，你得先去看看`user/ls.c`，还有`kernel/fs.h`的一部分，先看懂这几个再想要怎么做。

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/fs.h"
#include "user/user.h"
#include "kernel/fcntl.h"

char *
fmtname(char *path)
{
  char *p;

  // find the last '/' in the path
  for(p = path + strlen(path); p >= path && *p != '/'; --p)
    ;
  return p + 1;
}

void
find(char *path, char *filename)
{
  char buf[512], *p;
  int fd;
  struct stat st;
  struct dirent de;

  // get the file descriptor
  if((fd = open (path, 0)) < 0)
  {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  // get the file stats
  if(fstat (fd, &st) < 0)
  {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type)
  {
    case T_FILE: // if it is a file
      // check if the file name matches the given filename
      if(strcmp(fmtname(path), filename) == 0)
      {
        printf("%s\n", path);
      }
      break;

    case T_DIR: // if it is a directory
      strcpy(buf, path);
      p = buf + strlen(buf);
      *p++ = '/';
      
      // read the whole directory, until the end of the directory
      while(read(fd, &de, sizeof(de)) == sizeof(de))
      {
        if(de.inum == 0 || strcmp(de.name, ".") == 0
          || strcmp(de.name, "..") == 0)
          continue;

        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = 0;
        find(buf, filename); // recursive call to find in subdirectories
      }
      break;

    default:
      break;
  }
  close (fd);
}

int
main(int argc, char *argv[])
{
  if(argc < 3)
  {
    fprintf(2, "usage: find <start_path> <file_name>\n");
    exit(0);
  }
  find(argv[1], argv[2]);
  exit(0);
}
```
### 常见疑问
Q：为什么`p == path - 1`不会报错？      
A：因为没有解引用，只是存储那个地址是被允许的。

Q：`T_DIR`的那部分`while`循环是怎么运作的？     
A：每当你使用`read`成功读取了N个字节后，`fd`的文件偏移量就会增加N个字节。简单来说就是，读取成功后`fd`就会跳到下一个。       
ps：我自己也不是很清楚，写到这里时，我的进度只是LAB5: LAZY ALLOCATION而已，LAB9才是FILE SYSTEM，可能到时候就能理解了

## xargs (moderate)

这个稍微有一些难度，你得了解下管道，不用理解的很深，只要知道左边给右边传输数据，左边再读取数据就行了。

```C
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

#define READ 0

#define MAX_ARG_LEN 512
// #define MAXARG 32

int main(int argc, char *argv[])
{
    if (argc < 2) 
    {
        fprintf(2, "missing command\n");
        exit(1);
    }

    char line[MAX_ARG_LEN];
    int index = 0;
    char ch;
    
    // read a char
    while (read(READ, &ch, 1) > 0) 
    {
        // end of a line
        if (ch == '\n') 
        {
            line[index] = '\0'; // add null terminator
            
            // a memory per argument
            char *cmd_argv[MAXARG];
            for (int i = 0; i < argc - 1; i++) 
            {
                cmd_argv[i] = malloc(strlen(argv[i+1]) + 1);
                strcpy(cmd_argv[i], argv[i+1]); 
            }

            // a block for input line
            cmd_argv[argc-1] = malloc(strlen(line) + 1);
            strcpy(cmd_argv[argc-1], line);
            cmd_argv[argc] = 0; // add end of array

            int pid = fork();
            if (pid == 0) // child process
            {
                exec(cmd_argv[0], cmd_argv); // exec(path, argv)
                fprintf(2, "exec %s failed\n", cmd_argv[0]); // if exec fails
                exit(1);
            } 
            else if (pid > 0) // parent process
            {
                wait(0);
                
                // free memory allocated
                for (int i = 0; i < argc; i++) 
                {
                    free(cmd_argv[i]);
                }
            }
            else // fork failed 
            {
                fprintf(2, "fork failed\n");
                exit(1);
            }
            index = 0;    
        } 
        else // common char
        {
            // check if the argument length is out of limit
            if (index < MAX_ARG_LEN - 1) 
            {
                line[index++] = ch;
            } 
            else // argument too long
            {
                fprintf(2, "argument too long\n");
                while (read(READ, &ch, 1) > 0 && ch != '\n');
                index = 0;
            }
        }
    }
    exit(0);
}
```
### 常见疑问
Q：为什么`malloc`还要`+1`？   
A：记得数组吧？最后一个要放什么？没错，就是要放`0`/`NULL`用的。

Q：为什么程序一开始就`read`了，它读取的数据是什么？   
A：读取的数据是来自管道的，就是左边的`echo hello`，右边我们写`xargs`的就是读取+输出端，具体的我还不是很了解，还请各位自己去了解关于管道的知识！