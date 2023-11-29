# 模拟实现EXT2文件系统
## 实验目的
通过一个简单文件系统的设计，理解文件系统的内部结构、基本功能及实现。

## 实验内容
（1）分析EXT2文件系统的结构；

（2）基于Ext2的思想和算法，设计一个类Ext2的多级文件系统，实现Ext2文件系统的一个功能子集；

（3）用现有操作系统上的文件来代替硬盘进行硬件模拟。

## 实验步骤
ext2文件系统

![image](https://github.com/774512/test3/assets/148979339/e0d7c837-2639-43dc-b943-9b6cb53c7c45)

### 一、块的定义
逻辑块和物理块大小作为参考定义为512字节，每个组的数据块个数和inode数确定为4096

实验要求用现有操作系统上的文件来代替硬盘进行硬件模拟，该 “文件系统” 的所有用户信息、inode 信息、超级块信息、文件信息均以二进制方式保存在文件的特定地方

![image](https://github.com/774512/test3/assets/148979339/9b4df141-6f6e-4c0a-985e-c4606fa9928e)

一些常量：

```
#define VOLUME_NAME         "EXT2FS"    //卷名
#define BLOCK_SIZE          512         //块大小
#define DISK_SIZE           4612        //磁盘总块数

#define DISK_START          0           //磁盘开始地址
#define SB_SIZE             32          //超级块大小是32B

#define GD_SIZE             32          //块组描述符大小是32B
#define GDT_START           (0+512)     //块组描述符起始地址

#define BLOCK_BITMAP        (512+512)   //块位图起始地址 2*512
#define INODE_BITMAP        (1024+512)  //inode 位图起始地址 3*512

#define INODE_TABLE         (1536+512)  //索引节点表起始地址 4*512
#define INODE_SIZE          64          //每个inode的大小是 64B
#define INODE_TABLE_COUNTS  4096        //inode entry 数

#define DATA_BLOCK          264192      //数据块起始地址 4*512+4096*64
#define DATA_BLOCK_COUNTS   4096        //数据块数

#define BLOCKS_PER_GROUP    4612        //每组中的块数

#define USER_MAX            4           //用户个数
#define FOPEN_TABLE_MAX     16          //文件打开表大小
```
### 二、超级块
超级块(Super Block)描述整个分区的文件系统信息，如inode/block的大小、总量、使用量、剩余量，以及文件系统的格式与相关信息。

超级快数据结构：

```
struct super_block { // 32 B
	char sb_volume_name[16]; //文件系统名
   	unsigned short sb_disk_size; //磁盘总大小
   	unsigned short sb_blocks_per_group; // 每组中的块数
   	unsigned short sb_size_per_block;  // 块大小
   	char sb_pad[10];  //填充
};
```
合计32字节，占用一个块
### 三、组描述符
方便起见只设计一个组，每个块组描述符存储一个块组的描述信息，如在这个块组中从哪里开始是inode Table，从哪里开始是Data Blocks，空闲的inode和数据块还有多少个等等

组描述符数据结构

```
struct group_desc { // 32 B
    char bg_volume_name[16]; //文件系统名
    unsigned short bg_block_bitmap; //块位图的起始块号
    unsigned short bg_inode_bitmap; //索引结点位图的起始块号
    unsigned short bg_inode_table; //索引结点表的起始块号
    unsigned short bg_free_blocks_count; //本组空闲块的个数
    unsigned short bg_free_inodes_count; //本组空闲索引结点的个数
    unsigned short bg_used_dirs_count; //组中分配给目录的结点数
    char bg_pad[4]; //填充(0xff)
};
```



