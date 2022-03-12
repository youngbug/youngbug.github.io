---
layout: post_layout
title: ELF格式学习笔记 1.Object File
author: Zhao Yang(cnrgc@163.com)
time: 2020年07月10日 星期五
location: 北京
pulished: true
category: OS
tags: [operationsystem,reverse,linux]
excerpt_separator: "#"
---
这一部分描述了iABI object文件格式，被称为ELF（Exectable and Linking Format）。Object文件有三种主要类型：
* *重定位文件 relocatable file*中保存了代码和数据，用于和其他的目标文件链接来创建一个可执行文件或者共享目标文件。
* *可执行文件 executable file*保存用于执行的程序；该文件指出了*exec(BA_OS)*如何创建程序进程镜像(program's process imaging)。
* *共享目标文件 shared oject file*保存了代码和数据，用于链接再两个context中。第一，链接编辑器[参见ld(SD_CMD)]可以把它和其他重定位(relocatable)和共享目标文件(shared object file)处理创建成为另一个目标文件。第二，动态链接器组合它、其他可执行文件和其他共享目标文件来创建一个进程镜像(process image)。

# 文件格式 File Format
目标文件参与了程序的链接(构建一个程序)和程序的执行。为了方便和效率，目标文件(object file)格式提供了平行的视角。图1-1展示了目标文件(object file)的组织形式。


**图1-1： 目标文件格式Object File Format**

Linking View| 
----| 
ELF header| 
Program header table *optional*| 
Section 1 | 
...| 
Section n| 
...| 
...| 
Section header table| 

Execution View| 
----| 
ELF header| 
Program header table| 
Segment 1| 
Segment 2|
...| 
Section header table *optional*| 

*ELF header* 位于起始位置并且保存了描述文件组织的"road map"。*Section*保存着一块object file的信息，从链接视角看：指令、数据、符号表、重定位信息等。特殊section的描述会在*segments*和程序执行视角里提到。

如果存在*program header table*，告诉操作系统如何创建一个进程镜像(process image)。用于构建process image(execute a program)的文件，必须有program header table。重定位文件(relocatable file)不需要。*section header table*包含了描述文件section的信息。在这section header table中，每个section都有一个入口。每个入口给出了一些信息，包括section名称，section大小等。文件在链接时使用，必须有section header。其他object file可以有也可以没有。

>*Note:*  
尽管图1-1显示program header table就在ELF header后面，section header table跟着sections，实际的文件可能是跟图1-1不一样的。此外，sections和segment没有特别的顺序。只有ELF header的位置在文件中是固定的。

# 数据表示 Data Representation

这里描述的，object file *format*支持8-bit bytes和32-bit架构等多种处理器。尽管打算去支持更大(更小)的价格。因此object file的表示的控制数据是与机器无关的格式。

**图1-2： 32-Bit Data Types 32比特数据类型**

Name | Size | Alignment | Purpose
----|----|----|----
Elf32_Addr|4|4|Unsigned program address
Elf32_Half|2|2|Unsigned medium integer
Elf32_Off|4|4|Unsigned file offset
Elf32_Sword|4|4|Signed large integer
Elf32_Word|4|4|Unsigned large integer
unsigned char|1|1|unsigned small integer

object file定义的所有的数据结构都遵循自然大小和对齐的指导原则。如果需要，4字节的数据机构包含明确的填充以确保4字节对齐，强制数据结构大小是4的倍数。数据从文件的开始也有适当的对齐，比如，一个结构包含Elf32_Addr成员，将在文件中按4字节边界对齐。

因为移植的原因，ELF不适用位域(bit-fields)。

# ELF header

一些object file的控制结构可以增长，因为ELF header包含了他们的实际大小。如果object file format发生变化，程序可能遇到控制结构比预期的更大或者更小的情况。因此程序可能忽略"额外"的信息。"丢失"信息的处理依赖于上下文和扩展是否被定义。

**图1-3 ELF Header**

```c
/* The ELF file header.  This appears at the start of every ELF file.  */

#define EI_NIDENT (16)

typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf32_Half	e_type;			/* Object file type */
  Elf32_Half	e_machine;		/* Architecture */
  Elf32_Word	e_version;		/* Object file version */
  Elf32_Addr	e_entry;		/* Entry point virtual address */
  Elf32_Off	    e_phoff;		/* Program header table file offset */
  Elf32_Off	    e_shoff;		/* Section header table file offset */
  Elf32_Word	e_flags;		/* Processor-specific flags */
  Elf32_Half	e_ehsize;		/* ELF header size in bytes */
  Elf32_Half	e_phentsize;		/* Program header table entry size */
  Elf32_Half	e_phnum;		/* Program header table entry count */
  Elf32_Half	e_shentsize;		/* Section header table entry size */
  Elf32_Half	e_shnum;		/* Section header table entry count */
  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;
```
* **e_ident**   
最开始的字节标记这个文件是一个object file

* **e_type**   
这个成员定义了object file的类型

Name | Value | Meaning
----|----|----
ET_NONE|0|No file type
ET_REL|1|Relocatable file
ET_EXEC|2|Executable file
ET_DYN|3|Shared object file
ET_CORE|4|Core file
ET_LOPROC|0xFF00|Processor-specific
ET_HIPROC|0xFFFF|Processor-specific

尽管core file的内容是不特定的，类型ET_CORE是保留给这个文件的。值ET_LOPROC到ET_HIPROC是保留给processor-specific semantics。

* **e_machine**   
这个成员值指出这个文件需要的架构

Name | Value | Meaning
----|----|----
EM_NONE|0|No machine
EM_M32|1|AT&T WE 32100
EM_SPARC|2|SPARC
EM_386|3|Intel 80386
EM_68K|4|Motorola 68000
EM_88K|5|Motorola 88000
EM_860|7|Intel 80860
EM_MIPS|8|MIPS RS3000

其他值给新的机器类型保留。

* **e_version**   
识别object file版本

Name|Value|Meaning
----|----|----
EV_NONE|0|Invalid version
EV_CURRENT|1|Current version

* **e_entry**   
这个成员给出系统首次传输控制使用的虚拟地址，从此开始进程。如果文件没有入口点，这个成员的值保持为0.

* **e_phoff** 
这个成员保存着program header table在文件中偏移的字节值。如果文件没有program header table，这个值保持为0。

* **shoff**  
这个成员保存着section header table在文件中偏移的字节值。如果文件没有section header table，这个值保持为0。

* **e_flags**  
这个成员保存着和文件关联的processor-specific 标志位。标志位的名字采用*ET_machine_flag*的形式。

* **e_ehsize**  
这个成员保存了ELF header大小的字节数。

* **e_phentsize**  
这个成员保存了文件的program header table中的一个入口(entry)大小的字节数。所有的入口都是一样大的。

* **e_phnum**  
这个成员保存了program header table中入口(entry)的数量。因此*e_phentsize*和*e_phnum*的乘积给出了table大小的字节数。如果一个文件没有program header table，e_phnum保存0值。

* **e_shentsize**  
这个成员保存了一个section header大小的字节数。一个section header是section header table中的一个入口(entry)。所有的入口(entry)大小是一样的。

* **e_shnum**  
这个成员保存了section header table中的入口(entry)数量。因此*e_shentsize*和*e_shnum*的乘积给出了section header table大小的字节数。如果一个文件没有section header table，e_shnum的值为0。

* **e_shstrndx**  
这个成员保存了和section name string table关联的入口(entry)的section header table index。如果文件没有section name string table，这个成员的值为SHN_UNDEF。

# ELF Identification
上面提到了ELF提供的object file框架用于支持多种处理器，多种数据编码和多种机器类型。为了支持object file家族，文件的初始字节表明了如何解释这个文件，独立于处理器，和文件的剩下的部分无关。  
ELF header的初始字节符合**e_ident**成员。  
**图1-4：e_ident[] Identification Indexes**  

Name | Value | Purpose
----|----|----
EI_MAG0|0|File identification
EI_MAG1|1|File identification
EI_MAG2|2|File identification
EI_MAG3|3|File identification
EI_CLASS|4|File class
EI_DATA|5|Data encoding
ET_VERSION|6|File version
ET_PAD|7|Start of padding bytes
ET_NIDENT|16|Size of e_ident[]

通过index访问字节，这些字节的值如下：

* **EI_MAG0**到**EI_MAG3**   
文件的前4字节保存一个magic number，识别一个文件是不是ELF object file。  

Name | Value | Position
----|----|----
ELFMAG0|0x7F | e_ident[EI_MAG0]
ELFMAG1|'E'| e_ident[EI_MAG1]
ELFMAG2|'L'| e_ident[EI_MAG2]
ELFMAG3|'F'| e_ident[EI_MAG03]

* **EI_CLASS**  
下面的字节e_ident[EI_CLASS]，识别文件的class和capacity。  

Name | Value | Meaning 
----|----|----
ELFCLASSNONE|0|Invalid class
ELFCLASS32|1|32-bit objects
ELFCLASS64|2|64-bit objects

* **EI_DATA**  
字节e_ident[EI_DATA]明确了object file中process-specific数据的编码方式。下面的编码是现在定义的。  

Name | Value | Meaning
----|----|----
ELFDATANONE|0|Invalid data encoding
ELFDATA2LSB|1|后面定义
ELFDATA2MSB|2|后面定义

* **EI_VERSION**  
字节e_ident[EI_VERISON]明确了ELF header版本号。现在这个值必须为EV_CURRENT。

* **EI_PAD**  
这个值用来标记e_ident中没有使用的字节。这些字节保留并且被设置为0.程序读object file的时候会忽略这些字节。

**图1-5 Data Encoding** ELFDATA2LSB (Least Signifciant Bit)

Value | Byte 0 | Byte 1 | Byte 2 | Byte 3
----| ----|----|----|----|
0x01| 01
0x0102| 02|01
0x01020304|04|03|02|01

**图1-6 Data Encoding** ELFDATA2MSB (Most Signifciant Bit)

Value | Byte 0 | Byte 1 | Byte 2 | Byte 3
----| ----|----|----|----|
0x01| 01
0x0102| 01|02
0x01020304|01|02|03|04

# Machine Information
用于识别文件是32-bit Intel架构的e_ident，需要为如下取值。  

**图1-7：32-Bit Intel Architecture Identification,** e_ident  

Position | Value 
----|----
e_ident[EI_CLASS] | ELFCLASS32 
e_ident[EI_DATA] | ELFDATA2LSB 

处理器识别位于ELF header的e_machine成员，并且值必须为EM_386。
ELF header的e_flag成员保存和文件相关的bit标志位。32-bit Inter架构定义没有定义标志位，所以这个成员包含0。

# Sections
一个object file的section header table可以定位文件所有的sections。section header table是一个Elf32_Shdr结构的数组。一个section header table index是这个数组的下标。ELF header的e_shoff成员给出了从文件起始位置到section header table的偏移字节数。e_shum说明了section header table包含了多少入口(entry)。e_shentsize给出了每一个入口(entry)大小的字节数。

一些section header table的index是保留的。object file不能有这些特殊的index。

**图1-8:Special Section Indexes**

Name | Value
----|----
SHN_UNDEF|0
SHN_LORESERVE|0xFF00
SHN_LOPROC|0xFF00
SHN_HIPROC|0xFF1F
SHN_ABS|0xFFF1
SHN_COMMON|0xFFF2
SHN_HIRESERVE|0xFFFF

* **SHN_UNDEF**  
这个值标记未定义，丢失，不相关或者其他无意义的section参考。例如，一个符号"defined"相对于section编号SHN_UNDEF就是一个没有定义的符号。

> **NOTE:** 尽管下标索引0是保留给未定义的值，section header table包含一个索引0的入口。那是，如果ELF heder的e_shnum成员告诉系统在section header table这个文件有6个入口，他们有索引0~5。
