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
逻辑块和物理块大小作为参考定义为512字节，每个组的数据块个数和inode数确定为4096，每组容量确定为2MB

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
合计32字节，共占一个块

### 四、索引结点
一个文件除了数据需要存储之外，一些描述信息也需要存储，如文件类型，权限，文件大小，创建、修改、访问时间等，这些信息存在inode中而不是数据块中。inode表占多少个块在格式化时就要写入块组描述符中。 在Ext2文件系统中，每个文件在磁盘上的位置都由文件系统block group中的一个Inode指针进行索引，Inode将会把具体的位置指向一些真正记录文件数据的block块，需要注意的是这些block可能和Inode同属于一个block group也可能分属于不同的block group。我们把文件系统上这些真实记录文件数据的block称为Data blocks。

![image](https://github.com/774512/test3/assets/148979339/bfea0ef7-0ce2-4b69-b9f2-212d3107a4ed)

此模拟文件系统共4096个数据块，采用了多级索引机制，定义直接索引结点6个，一级子索引定义1个，二级子索引定义1个

```
struct inode { // 64 B = 6 * 2B + 5 * 4B + 16B + 16B
    unsigned short i_mode;   //文件类型及访问权限
    unsigned short i_blocks; //文件所占的数据块个数(0~7), 最大为7
    unsigned short i_uid;    //文件拥有者标识号
    unsigned short i_gid;    //文件的用户组标识符
    unsigned short i_links_count; //文件的硬链接计数
    unsigned short i_flags;  //打开文件的方式
    unsigned long i_size;    //文件或目录大小(单位 byte)
    unsigned long i_atime;   //访问时间
    unsigned long i_ctime;   //创建时间
    unsigned long i_mtime;   //修改时间
    unsigned long i_dtime;   //删除时间
    unsigned short i_block[8]; //指向数据块号
    char i_pad[16];           //填充(0xff)
};
```

### 五、目录与文件
目录作为特殊文件处理。第一个索引结点指向根目录，根目录的索引结点中直接索引域指向数据块0

```
struct dir_entry { // 16 B
    unsigned short inode; //索引节点号
    unsigned short rec_len; //目录项长度
    unsigned short name_len; //文件名长度
    char file_type; //文件类型(1 普通文件 2 目录.. )
    char name[9]; //文件名
};
```

当文件系统在初始化时，根目录的数据块(即数据块1)将被初始化。其所包含的所有索引节点号以及目录项长度域将被置0。当文件被删除时，其所在目录项长度不变，索引节点号将被置0。

当新建一个文件时，程序将从目录的数据块查找索引节点号为0的目录项，并检查其长度是否够用。是,则分配给该文件，否则继续查找，直到找到长度够用，或者是长度为0 (即未被使用过)的地址，为文件建立目录项。

当建立的是一个目录时，将其所分配到的索引结点所指向的数据块清空。并且自动写入两个特殊的目录项。一个是当前目录“.”，其索引结点即指向本身的数据块。另一个是上一级目录“..”，其索引结点指向上一级目录的数据块。
### 六、用户信息
```
struct user {
    char username[10];
    char password[10];
    unsigned short u_uid;   //用户标识号
    unsigned short u_gid;
}User[USER_MAX];
```
### 七、文件系统实现的功能函数
```
extern void initialize_user();  //初始化用户
extern int login(char username[10], char password[10]); //用户登录
extern void initialize_memory(); //初始化内存
extern void format(); //格式化文件系统
extern void cd(char tmp[100]); //进入某个目录，实际上是改变当前路径
extern void mkdir(char tmp[100], int type); //创建目录
extern void cat(char tmp[100], int type); //创建文件
extern void rmdir(char tmp[100]); //删除一个空目录
extern void del(char tmp[100]); //删除文件
extern void open_file(char tmp[100]); //打开文件
extern void close_file(char tmp[100]); //关闭文件
extern void read_file(char tmp[100]); //读文件内容
extern void write_file(char tmp[100]); //文件以覆盖方式写入
extern void ls(); //查看目录下的内容
extern void check_disk(); //检查磁盘状态
extern void help(); //查看指令
extern void chmod(char tmp[100], unsigned short mode); //修改文件权限
```
### 八、一些读写方法的处理
进行超级块、组描述符、位图、inode 节点、数据块的读写时，需要用到一些缓冲区，定义如下：

```
static struct super_block sb_block[1];  // 超级块缓冲区
static struct group_desc gdt[1];  // 组描述符缓冲区
static struct inode inode_area[1]; // inode缓冲区
static unsigned char bitbuf[512] = {0}; // block位图缓冲区
static unsigned char ibuf[512] = {0}; // inode位图缓冲区
static struct dir_entry dir[32];  // 目录项缓冲区 32*16=512
static char Buffer[BLOCK_SIZE]; // 针对数据块的缓冲区
```

（1）读写超级块

```
// 写超级块
static void update_super_block() {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, DISK_START, SEEK_SET);
    fwrite(sb_block, SB_SIZE, 1, fp);
    fflush(fp); //立刻将缓冲区的内容输出，保证磁盘内存数据的一致性
}

// 读超级块
static void reload_super_block() {
    fseek(fp, DISK_START, SEEK_SET);
    fread(sb_block, SB_SIZE, 1, fp);//读取内容到超级块缓冲区中
}
```

先定位文件内部位置指针的位置，再进行块的读写，其中定位文件内部位置指针用到的库函数为 int fseek(FILE *stream, long offset, int fromwhere)。同时在写完后用库函数 int fflush(FILE *stream) 清除读写缓冲区。

（2）读写组描述符

```
// 写组描述符
static void update_group_desc() {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, GDT_START, SEEK_SET);
    fwrite(gdt, GD_SIZE, 1, fp);
    fflush(fp);
}

// 读组描述符
static void reload_group_desc() {
    fseek(fp, GDT_START, SEEK_SET);
    fread(gdt, GD_SIZE, 1, fp);
}
```

（3）读写块位图

```
// 写 block 位图
static void update_block_bitmap() {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, BLOCK_BITMAP, SEEK_SET);
    fwrite(bitbuf, BLOCK_SIZE, 1, fp);
    fflush(fp);
}

// 读 block 位图
static void reload_block_bitmap() {
    fseek(fp, BLOCK_BITMAP, SEEK_SET);
    fread(bitbuf, BLOCK_SIZE, 1, fp);
}
```

（4）读写inode位图

```
// 写 inode 位图
static void update_inode_bitmap() {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, INODE_BITMAP, SEEK_SET);
    fwrite(ibuf, BLOCK_SIZE, 1, fp);
    fflush(fp);
}

// 读 inode 位图
static void reload_inode_bitmap() {
    fseek(fp, INODE_BITMAP, SEEK_SET);
    fread(ibuf, BLOCK_SIZE, 1, fp);
}
```

（5）读写第i个inode

```
// 写第 i 个 inode
static void update_inode_entry(unsigned short i) {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, INODE_TABLE + (i - 1) * INODE_SIZE, SEEK_SET);
    fwrite(inode_area, INODE_SIZE, 1, fp);
    fflush(fp);
}

// 读第 i 个 inode
static void reload_inode_entry(unsigned short i) {
    fseek(fp, INODE_TABLE + (i - 1) * INODE_SIZE, SEEK_SET);
    fread(inode_area, INODE_SIZE, 1, fp);
}
```

（6）读写第i个数据块

```
// 写第 i 个数据块
static void update_dir(unsigned short i) {
    fp = fopen("./Ext2", "rb+");
    fseek(fp, DATA_BLOCK + i * BLOCK_SIZE, SEEK_SET);
    fwrite(dir, BLOCK_SIZE, 1, fp);
    fflush(fp);
}

// 读第 i 个数据块
static void reload_dir(unsigned short i) {
    fseek(fp, DATA_BLOCK + i * BLOCK_SIZE, SEEK_SET);
    fread(dir, BLOCK_SIZE, 1, fp);
} 
```
### 九、内存的分配，查找，删除
（1）分配数据块

```
// 分配一个数据块,返回数据块号
static int alloc_block() {
    //bitbuf共有512个字节，表示4096个数据块。根据last_alloc_block/8计算它在bitbuf的哪一个字节
    unsigned short cur = last_alloc_block;
    unsigned char con = 128; // 1000 0000b
    int flag = 0;
    if (gdt[0].bg_free_blocks_count == 0) {
        printf("There is no block to be allocated!\n");
        return (0);
    }
    reload_block_bitmap();
    cur /= 8;
    while (bitbuf[cur] == 255) { //该字节的8个bit都已有数据
        if (cur == 511)
            cur = 0; //最后一个字节也已经满，从头开始寻找
        else
            cur++;
    }
    while (bitbuf[cur] & con) { //在一个字节中找具体的某一个bit
        con = con / 2;
        flag++;
    }
    bitbuf[cur] = bitbuf[cur] + con;
    last_alloc_block = cur * 8 + flag;

    update_block_bitmap();
    gdt[0].bg_free_blocks_count--;
    update_group_desc();
    return last_alloc_block;
}
```

（2）分配inode

```
// 分配一个 inode
static int get_inode() {
    unsigned short cur = last_alloc_inode;
    unsigned char con = 128;
    int flag = 0;
    if (gdt[0].bg_free_inodes_count == 0) {
        printf("There is no Inode to be allocated!\n");
        return 0;
    }
    reload_inode_bitmap();

    cur = (cur - 1) / 8;   //第一个标号是1，但是存储是从0开始的
    while (ibuf[cur] == 255) { //先看该字节的8个位是否已经填满
        if (cur == 511)
            cur = 0;
        else
            cur++;
    }
    while (ibuf[cur] & con) { //再看某个字节的具体哪一位没有被占用
        con = con / 2;
        flag++;
    }
    ibuf[cur] = ibuf[cur] + con;
    last_alloc_inode = cur * 8 + flag + 1;
    update_inode_bitmap();
    gdt[0].bg_free_inodes_count--;
    update_group_desc();
    return last_alloc_inode;
}
```

（3）查找文件或目录

```
// 查找当前目录中名为tmp的文件或目录，并得到该文件的inode号，它在上级目录中的数据块号以及数据块中目录的项号
static unsigned short research_file(char tmp[100], int file_type, unsigned short *inode_num,
                                   unsigned short *block_num, unsigned short *dir_num) {
    unsigned short j, k;
    reload_inode_entry(current_dir); //进入当前目录
    j = 0;
    while (j < inode_area[0].i_blocks) {
        reload_dir(inode_area[0].i_block[j]);
        k = 0;
        while (k < 32) {
            if (!dir[k].inode || dir[k].file_type != file_type || strcmp(dir[k].name, tmp)) {
                k++;
            }
            else {
                *inode_num = dir[k].inode;
                *block_num = j;
                *dir_num = k;
                return 1;
            }
        }
        j++;
    }
    return 0;
}
```

（4）为新增目录或文件分配 dir_entry

```
// 为新增目录或文件分配 dir_entry
// 对于新增文件，只需分配一个 inode 号
// 对于新增目录，除了 inode 号外，还需要分配数据区存储.和..两个目录项
static void dir_prepare(unsigned short tmp, unsigned short len, int type) {
    reload_inode_entry(tmp);

    if (type == 2) { // 目录
        inode_area[0].i_size = 32;
        inode_area[0].i_blocks = 1;
        inode_area[0].i_block[0] = alloc_block();
        dir[0].inode = tmp;
        dir[1].inode = current_dir;
        dir[0].name_len = len;
        dir[1].name_len = current_dirlen;
        dir[0].file_type = dir[1].file_type = 2;

        for (type = 2; type < 32; type++)
            dir[type].inode = 0;
        strcpy(dir[0].name, ".");
        strcpy(dir[1].name, "..");
        update_dir(inode_area[0].i_block[0]);

        inode_area[0].i_mode = 01006;
    }
    else {
        inode_area[0].i_size = 0;
        inode_area[0].i_blocks = 0;
        inode_area[0].i_mode = 0407;
    }
    update_inode_entry(tmp);
}
```

（5）删除一个块

```
// 删除一个块
static void remove_block(unsigned short del_num) {
    unsigned short tmp;
    tmp = del_num / 8;
    reload_block_bitmap();
    switch (del_num % 8) { // 更新block位图 将具体的位置为0
        case 0:
            bitbuf[tmp] = bitbuf[tmp] & 127;
            break; // bitbuf[tmp] & 0111 1111b
        case 1:
            bitbuf[tmp] = bitbuf[tmp] & 191;
            break; //bitbuf[tmp]  & 1011 1111b
        case 2:
            bitbuf[tmp] = bitbuf[tmp] & 223;
            break; //bitbuf[tmp]  & 1101 1111b
        case 3:
            bitbuf[tmp] = bitbuf[tmp] & 239;
            break; //bitbuf[tmp]  & 1110 1111b
        case 4:
            bitbuf[tmp] = bitbuf[tmp] & 247;
            break; //bitbuf[tmp]  & 1111 0111b
        case 5:
            bitbuf[tmp] = bitbuf[tmp] & 251;
            break; //bitbuf[tmp]  & 1111 1011b
        case 6:
            bitbuf[tmp] = bitbuf[tmp] & 253;
            break; //bitbuf[tmp]  & 1111 1101b
        case 7:
            bitbuf[tmp] = bitbuf[tmp] & 254;
            break; // bitbuf[tmp] & 1111 1110b
    }
    update_block_bitmap();
    gdt[0].bg_free_blocks_count++;
    update_group_desc();
}
```

（6）删除一个inode

```
// 删除一个 inode
static void remove_inode(unsigned short del_num) {
    unsigned short tmp;
    tmp = (del_num - 1) / 8;
    reload_inode_bitmap();
    switch ((del_num - 1) % 8) { //更改block位图
        case 0:
            bitbuf[tmp] = bitbuf[tmp] & 127;
            break;
        case 1:
            bitbuf[tmp] = bitbuf[tmp] & 191;
            break;
        case 2:
            bitbuf[tmp] = bitbuf[tmp] & 223;
            break;
        case 3:
            bitbuf[tmp] = bitbuf[tmp] & 239;
            break;
        case 4:
            bitbuf[tmp] = bitbuf[tmp] & 247;
            break;
        case 5:
            bitbuf[tmp] = bitbuf[tmp] & 251;
            break;
        case 6:
            bitbuf[tmp] = bitbuf[tmp] & 253;
            break;
        case 7:
            bitbuf[tmp] = bitbuf[tmp] & 254;
            break;
    }
    update_inode_bitmap();
    gdt[0].bg_free_inodes_count++;
    update_group_desc();
}
```

（7）在打开文件表中查找是否已打开某个文件

```
// 在打开文件表中查找是否已打开文件
static unsigned short search_file(unsigned short Inode) {
    unsigned short fopen_table_point = 0;
    while (fopen_table_point < 16 && fopen_table[fopen_table_point++] != Inode);
    if (fopen_table_point == 16) {
        return 0;
    }
    return 1;
}
```
## 运行结果
使用命令 gcc main.c init.c –o ext2 进行编译链接：






