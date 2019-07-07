---
title: mach-O file format
date: 2019-07-07 23:58:54
tags: mach-O
---
##  
mach-O 文件是macOS上的二进制文件格式，相当于windows上的PE格式 跟 linux上的elf文件格式，macOS上的主要的可执行文件都是这种格式，今天就分析下mach-O的文件格式
## Format

mach-O文件主要分为下面三部分:
* mach header
* load commands
* segments(Data)
  
![avatar](https://raw.githubusercontent.com/4nalysis/4nalysis.github.io/hexo/source/res/mach-O-file-format/macho.jpg)
  
如上图所示。同时在xnu中也有对于这种格式的一些结构定义，下面从结构上分析一下。

### macho header

macho header中保存了文件的magic、cputype 、文件大小等一些基本信息。对于更细节的信息，可以参考xnu 源代码中，对于macho header的定义。

```
/*
* The 32-bit mach header appears at the very beginning of the object file for
* 32-bit architectures.
*/
struct mach_header {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu specifier */
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* type of file */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
};

/* Constant for the magic field of the mach_header (32-bit architectures) */
#define MH_MAGIC    0xfeedface  /* the mach magic number */
#define MH_CIGAM    0xcefaedfe  /* NXSwapInt(MH_MAGIC) */

/*
* The 64-bit mach header appears at the very beginning of object files for
* 64-bit architectures.
*/
struct mach_header_64 {
    uint32_t    magic;      /* mach magic number identifier */
    cpu_type_t  cputype;    /* cpu type */ x86_64  i386 arm 等等
    cpu_subtype_t   cpusubtype; /* machine specifier */
    uint32_t    filetype;   /* 文件类型 */
    uint32_t    ncmds;      /* number of load commands */
    uint32_t    sizeofcmds; /* the size of all the load commands */
    uint32_t    flags;      /* flags */
    uint32_t    reserved;   /* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */ 大端序
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) * 小端序，这里说明一下，在ios设备上是小端序
```

主要参数:

magic:可以文件的标志，可以识别出文件是32位还是64位 同时也可以区分文件时大端还是小端\
cputype:标识cpu的架构，比如i386、x86_64、arm等\
filetype:下面列出一些主要的filetype

|FILETYPE|a|
|---|-- |
|MH_OBJECT|编译过程产生的目标文件 .o|
|MH_EXECUTE|可执行文件 |
|MH_CORE| core dump |
|MH_DYLIB|动态库 |
|MH_KEXT_BUNDLE| 内核扩展文件 |
|MH_DYLINKER| 调试符号文件 |

### Load commands
紧接着header 后面就是load commands的部分，由n个load_command的结构组成，数量跟大小在header里面可以获取到。

```
struct load_command {
    uint32_t cmd;       /* type of load command */
    uint32_t cmdsize;   /* total size of command in bytes */
};
```
  对于不同类型的load_command 可以用不同的结构体来读取后面的数据，主要的一些类型:

  |||
  |--| -- |
  |LC_SEGMENT||
  |LC_UNIXTHREAD|unix 线程|
  |LC_MAIN|主线程|
  |LC_UUID|唯一ID|
  |LC_LOAD_DYLINKER|连接器|
  |LC_LOAD_DYLIB|依赖的动态库|

ps:从Mountain Lion开始，一条新的加载命令LC_MAIN替代了LC_UNIXTHREAD命令。这条命令的作用是设置程序主线程的入口点地址和大小。这条命令比LC_UNIXTHREAD命令更实用一个些，所以后来的macho 不一定兼容之前的系统。

最多的结构是LC_SEGMENT
```
struct segment_command { /* for 32-bit architectures */
    uint32_t    cmd;        /* LC_SEGMENT */
    uint32_t    cmdsize;    /* includes sizeof section structs */
    char        segname[16];    /* segment name */
    uint32_t    vmaddr;     /* memory address of this segment */
    uint32_t    vmsize;     /* memory size of this segment */
    uint32_t    fileoff;    /* file offset of this segment */
    uint32_t    filesize;   /* amount to map from the file */
    vm_prot_t   maxprot;    /* maximum VM protection */
    vm_prot_t   initprot;   /* initial VM protection */
    uint32_t    nsects;     /* number of sections in segment */
    uint32_t    flags;      /* flags */
};


struct segment_command_64 { /* for 64-bit architectures */
    uint32_t    cmd;        /* LC_SEGMENT_64 */
    uint32_t    cmdsize;    /* includes sizeof section_64 structs */
    char        segname[16];    /* segment name */
    uint64_t    vmaddr;     /* memory address of this segment */
    uint64_t    vmsize;     /* memory size of this segment */
    uint64_t    fileoff;    /* file offset of this segment */
    uint64_t    filesize;   /* amount to map from the file */
    vm_prot_t   maxprot;    /* maximum VM protection */
    vm_prot_t   initprot;   /* initial VM protection */
    uint32_t    nsects;     /* number of sections in segment */
    uint32_t    flags;      /* flags */
};
```
segment_command 结构体中保存着结构体的基本信息，包括在文件中偏移地址,占用文件大小，虚拟地址，虚拟地址大小，用于解析macho文件，像一个索引一样，真正的指向的数据在下一部分的data部分。

### Data & Segment
data 部分分为不同的segment  其中segment 可能包含多个section，是存放主要数据的地方，前面segment的数据都指向这里。section的结构会帮助内核将数据映射到虚拟内存。

```
struct section { /* for 32-bit architectures */
    char        sectname[16];   /* name of this section */
    char        segname[16];    /* segment this section goes in */
    uint32_t    addr;       /* memory address of this section */
    uint32_t    size;       /* size in bytes of this section */
    uint32_t    offset;     /* file offset of this section */
    uint32_t    align;      /* section alignment (power of 2) */
    uint32_t    reloff;     /* file offset of relocation entries */
    uint32_t    nreloc;     /* number of relocation entries */
    uint32_t    flags;      /* flags (section type and attributes)*/
    uint32_t    reserved1;  /* reserved (for offset or index) */
    uint32_t    reserved2;  /* reserved (for count or sizeof) */
};

struct section_64 { /* for 64-bit architectures */
    char        sectname[16];   /* name of this section */
    char        segname[16];    /* segment this section goes in */
    uint64_t    addr;       /* memory address of this section */
    uint64_t    size;       /* size in bytes of this section */
    uint32_t    offset;     /* file offset of this section */
    uint32_t    align;      /* section alignment (power of 2) */
    uint32_t    reloff;     /* file offset of relocation entries */
    uint32_t    nreloc;     /* number of relocation entries */
    uint32_t    flags;      /* flags (section type and attributes)*/
    uint32_t    reserved1;  /* reserved (for offset or index) */
    uint32_t    reserved2;  /* reserved (for count or sizeof) */
    uint32_t    reserved3;  /* reserved */
};
```
|||
|-|-|
|__text|代码|
|__data|the real initialized data section|
|__cstring|硬编码的字符串|
|-|-|


## 小结
理解macho的结构，便于后面理解macho文件在系统的加载过程，同时对于一些样本分析 以及相对底层的安全开发也有帮助。