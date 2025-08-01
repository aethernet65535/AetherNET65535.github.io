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
- 存储地址，指向一个「索引块」
- 索引块里有256个数据块地址

二级索引：
- 存储地址，指向一个「集合块」
- 集合块存储着256个「一级索引块」
- 每个一级索引块能存储256个数据块地址
- 所以一个二级索引最多能存储256×256个块，也就是65536个

差不多是这样：      
> 一级索引 -> 256个块地址       
> 二级索引 -> 256个一级索引地址 -> (256*256)个块地址        

如果算总空间的话...一开始是256+12，也就是268×1024，274,432B、268kB，说实话好像有点小。    
现在的话，是(11)+(256)+(256×256)，也就是65,804kB、64mB，那确实挺多的了。

## 修改inode结构
```C
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};
```

```C
```

最后编辑时间：2025/7/28