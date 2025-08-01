+++
title = "XV6-2020 LAB9-FILESYSTEM"
date = 2025-07-21

[taxonomies]
tags = ["xv6"]

[extra]
comment = true
+++

# 前言
这个lab能学到的东西还挺多的，算是一个很好的文件系统入门了。     

# large files (moderate)
第一个小作业是要给xv6现有的fs添加一个二级索引，我们先说说原本的是怎么样的吧。       
> [0 - 11] 直接索引 | [12] 一级索引
然后我们要改成这样：        
> [0 - 10] 直接索引 | [11] 一级索引 | [12] 二级索引

直接索引：
- 存储指向块的地址
- 能直接找到数据，不需要在跳转

一级索引：
- 存储地址，指向一个**「索引块」**
- 索引块里有**256个**数据块地址

二级索引：
- 存储地址，指向一个**「集合块」**
- 集合块存储着**256个「一级索引块」**
- 每个一级索引块能存储**256个**数据块地址
- 所以一个二级索引最多能存储**256×256个**块，也就是**65536个**

差不多是这样：      
> 一级索引 -> 256个块地址       
> 二级索引 -> 256个一级索引地址 -> (256*256)个块地址        

如果算总空间的话...一开始是256+12，也就是268×1024，274,432B、**268kB**，说实话好像有点小。    
现在的话，是(11)+(256)+(256×256)，也就是65,804kB、**64mB**，那确实挺多的了。

## 修改inode结构
根据xv6的提示，我们需要把`NDIRECT`改为**11**，因为之前的其中一个要用来放**二级索引**：
```C
#define NDIRECT 11 // 直接块
#define DINDIRECT (NDIRECT + 1) // 二级索引
#define NINDIRECT (BSIZE / sizeof(uint)) // 一级索引能存的地址量
#define MAXFILE (NDIRECT + NINDIRECT + (NINDIRECT * NINDIRECT))
```

然后`inode`结构体的`addrs`字段就要改为**直接块（0-10）、一级索引（11）、二级索引（12）**：
```C
struct dinode {
  short type;               // File type
  short major;              // Major device number (T_DEVICE only)
  short minor;              // Minor device number (T_DEVICE only)
  short nlink;              // Number of links to inode in file system
  uint size;                // Size of file (bytes)
  uint addrs[NDIRECT+2];    // Data block addresses
};
```

## 修改bmap函数
### 修改前
这个函数的作用是：    
返回：在实参inode结构体中，对应块号的地址   
如果没有这个块号的地址，那就创建一个。

这个函数有点难，所以我们慢慢看。    
首先，第一段：
```C
// 直接块处理方法
if(bn < NDIRECT){ 
    if((addr = ip->addrs[bn]) == 0) 
      ip->addrs[bn] = addr = balloc(ip->dev);  
    return addr;
  }
```
代码作用：    
1. 如果bn下标还没被占用，那就分配一个空闲块号。    
2. 如果里面已经有地址了，那直接赋值给addr，然后这函数基本就完工了。

其次，第二段：
```C
// 这个很重要的，注意
// 没有这个的话那你块号是255之类的就完啦！
// 或者其他合法但是在这里过大的索引
// 不是错位就是爆炸！
bn -= NDIRECT;

// 一级索引处理方法
if(bn < NINDIRECT){
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
```
代码作用：    
1. 如果索引块还没创建，那就分配一个索引块
2. 获取该数据块的缓存
3. 对该缓存块的`data`字段进行类型转换并赋值给`a`    
> `uchar data[1024]`在转换后，就会变成`uint*[256]`了    
> 你可能会疑惑“啊？为什么不是**128**呢？**1024/8**是**128**呀！   
> 但是其实这里是用`uint`本身的大小转换的呢，很神奇吧，就是**4字节**！   
> 反正就是这样，明白不是用指针大小转换就行了！    

4. 如果在这个256个地址的索引块数组内没有对应bn的地址，那就分配一块空闲的
> 然后这里还要调用`log_write()`，具体原因博主不是很清楚，
似乎是因为索引块并不是直接属于inode的，所以就得调用   
> 别忘了调用`brelse()`释放锁，因为`bread()`会调用`bget()`，这过程会让对应的块缓存被锁，
所以一定得释放锁，毕竟我们已经好了
5. 如果啥事没有的话，就直接给用户返回addr

### 修改后
基本照葫芦画瓢就可以做出一个**二级索引VER**了呢！   
```C
bn -= NINDIRECT;

if(bn < NINDIRECT * NINDIRECT){ // 只要在65535范围内的就都是合法bn
  dbn = bn / NINDIRECT; // 在哪个一级索引块？
  dbnoff = bn % NINDIRECT; // 在该块的第几个索引？

  if((addr = ip->addrs[DINDIRECT]) == 0)
    ip->addrs[DINDIRECT] = addr = balloc(ip->dev);

  // 和刚刚的一样，不过读者可以把它想得更简单点
  // 想成“进入该块”就可以了，这次我们进入的是集合块
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
    
  // 在集合块内有没有对应的一级索引块？
  if((addr2 = a[dbn]) == 0){
    a[dbn] = addr2 = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);

  // 这次进入的是对应的一级索引块
  bp2 = bread(ip->dev, addr2);
  a2 = (uint*)bp2->data;

  // 在该一级索引块有没有对应bn的块号？
  if((addr = a2[dbnoff]) == 0){
    a2[dbnoff] = addr = balloc(ip->dev);
    log_write(bp2);
  }
  brelse(bp2);
  
  return addr;
}
```



最后编辑时间：2025/8/1