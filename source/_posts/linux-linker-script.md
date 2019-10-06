---
title: GNU 链接脚本LDS介绍
tags:
  - LD Script
  - Linux
  - 链接器
url: 1684.html
id: 1684
categories:
  - Linux
  - Linux编译
date: 2019-01-06 22:38:44
---

前言
--

程序的从C语言代码变成可以在目标机器上执行额文件，可以分为如下步骤

*   编译
    *   预编译：将宏定义等转义编译：将C语言变成目标文件(.o档案)
    *   编译/汇编：将预编译过后的目标变为目标文件
*   链接：合并多个目标文件(.o/.a)等为最终的可执行文件。

LD命令是GNU链接程序，它可以接受 _ld -T_ 输入链接脚本，根据链接脚本的定义来决定链接方式。在 **[Linux中断(2)](https://l2h.site/2019/01/05/linux-interrupt-4/)** 一文中，有简单提到过Linux里用到了很多链接技巧。因此，学习Linux内核，多少需要对链接脚本基本语法格式有初步了解。 本文即对GNU链接脚本的基本格式进行一些介绍。

先看一个基本的链接脚本。在执行_ld_链接时，如果不传入链接脚本，会使用默认的链接脚本。_ld --verbose_可以显示默认的链接脚本，如下：
```
[lambert ~]$ ld --verbose
GNU ld version 2.25.1-32.base.el7_4.1
  Supported emulations:
   elf_x86_64
   elf32_x86_64
   elf_i386
   i386linux
   elf_l1om
   elf_k1om
using internal linker script:
==================================================
/* Script for -z combreloc: combine and sort reloc sections */
/* Copyright (C) 2014 Free Software Foundation, Inc.
   Copying and distribution of this script, with or without modification,
   are permitted in any medium without royalty provided the copyright
   notice and this notice are preserved.  */
OUTPUT_FORMAT("elf64-x86-64", "elf64-x86-64",
              "elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)
SEARCH_DIR("/usr/x86_64-redhat-linux/lib64"); SEARCH_DIR("/usr/lib64"); SEARCH_D                                                                                                                                                             IR("/usr/local/lib64"); SEARCH_DIR("/lib64"); SEARCH_DIR("/usr/x86_64-redhat-lin                                                                                                                                                             ux/lib"); SEARCH_DIR("/usr/local/lib"); SEARCH_DIR("/lib"); SEARCH_DIR("/usr/lib                                                                                                                                                             ");
SECTIONS
{
  /* Read-only sections, merged into text segment: */
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SE                                                                                                                                                             GMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
  .interp         : { *(.interp) }
  .note.gnu.build-id : { *(.note.gnu.build-id) }
  .hash           : { *(.hash) }
  .gnu.hash       : { *(.gnu.hash) }
  .dynsym         : { *(.dynsym) }
  .dynstr         : { *(.dynstr) }
  .gnu.version    : { *(.gnu.version) }
  .gnu.version_d  : { *(.gnu.version_d) }
  .gnu.version_r  : { *(.gnu.version_r) }
/* 以下部分省略 */
```
基本概念
----

### **SECTION**

我们知道一个可执行文件有多个段，如.text段、BSS段、data段等。这里SECTION即对应可执行程序的段。一个段由以下几个部分组成：

*   段名称
*   段内容
*   段长度信息

段可以定义为Loadable和allocatable两种加载方式

*   loadable: 执行时该section是否需要被加载到内存
*   allocatable: 先保留内存的一块空间让程序执行时使用，如.bss段

### Symbol

一个object 档案存放多个symbol，又称为symbol table（符号表）。Symbol通常就是全局变量、静态变量或是函数的名称。我们可以使用_**objdump -t 可执行文件**_ 或者 _**readelf -a 可执行文件**_或者_**nm 可执行文件**_来查看对应可执行文件的符号表。

### LMA V.S. VMA

LMA: Load Memory Address, 表示程序被装载到内存哪个位置

VMA: Virutal Memroy Address，表示程序执行的位置，即CPU执行对应程序指令的位置。

通常，LMA = VMA。但是在一些嵌入式体系中， LMA和VMA不一样。而其中最常见的一种情况就是，程序被放到ROM中。而程序要运行时候的地址是内存虚拟地址，也就是VMA。之后小节会有例子做介绍。

Link Script语法
-------------

Link Script可以基本总结如下：

*   以文本形式存放
*   由多个command組成
*   每个command可能是
    *   keyword + 参数
    *   设定symbol
    *   …
*   command 可以用`;`分開，空白會被忽略
*   使用/_* * _/注释
*   字串直接打，如果有用到script保留的字元如`.`可以用`"`包住

### 简单链接脚本范例
```
SECTIONS
{
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```
**_._**表示内存位置，起始值为0。结束值则由链接器计算把所有input section的数据整合到output section的长度。而.如果没有指定明确的内存地址的话，就会被设定为上一个地址的结束地址。

第三行：设定内存位置为0x10000

第四行：把所有**输入**object档中({ *(.text) })存放到**输出**object档案的.text区块中。

第五行：设定内存位置为0x8000000

第六行、第七行：先放已初始化全局变量（.data），所有输入目标文件中的.data字段都会被打包到此位置；紧接着再放未初始值的全局变量（.bss）。

### 简单链接脚本命令

_**ENTRY(symbol)**_ ：设置程序入口点

#### 文件相关命令

**_INCLUDE filename  
_**在看到这个命令的时候才去载入filename这个linker script。可以被放在不同的命令如SETCTION, MEMORY等。  
_**INPUT(file1 file2 …)  
**_指定加载的输入object档案，如abc.o这样的档案。  
_**GROUP(file1 file2 …)  
**_指定加载的输入archieve档案，如libabc.a这样的档案。  
_**AS_NEEDED(file1 file2 …)  
**_在INPUT和GROUP使用的命令，用来告诉linker说如果object里面的数据有被reference到才link进来，猜测应该可以减少储存空间。范例（未测试请自行斟酌）：INPUT(file1.o file2.o AS_NEEDED(file3.o file4.o))  
_**OUTPUT(filename)  
**_和gcc -o filename 一样  
**_SEARCH_DIR(path)  
_**和-L path一样  
_**STARTUP(filename)  
**_和INPUT相同，唯一差别是ld保证这个档案一定是第一个被link  

#### 输出文件格式相关命令

_**OUTPUT_FORMAT(bfdname)  
**_指定输出object档案的binary 文件格式，可以使用objdump -i列出支持的binary 文件格式  
_**OUTPUT_FORMAT(default, big, little)  
**_指定输出object档案预设的binary 文件格式，big endian的binary 文件格式以及little endian的binary 文件格式。可以使用objdump -i列出支持的binary 文件格式  
_**TARGET(bfdname)  
**_告诉ld用那种binary 文件格式读取输入object档案要，可以使用objdump -i列出支持的binary 文件格式

#### 设定内存块别名的命令

**_REGION_ALIAS(alias, region)_** 对内存区域设定别名，这样输出段可以灵活地映射。具体可以参考<[LD:REGION_ALIAS](https://sourceware.org/binutils/docs/ld/REGION_005fALIAS.html#REGION_005fALIAS)>

#### 其他链接器相关命令

**_ASSERT(exp, message)_** 条件不成立打印message并结束链接过程  
_**EXTERN(symbol1 symbol2 ...)**_ 强迫让指定的symbol设成undefined，手册说一般用在刻意要使用非标准的API。  
_**FORCE_COMMON_ALLOCATION**_ 手册和男人说和兼容性有关，手册上是说强迫分配空间给common symbols，即使是link relocate档案。(common symbols不知道是什么)  
_**OUTPUT_ARCH(bfdarch)**_ 指定输出的平台，可以透过objdump -i查询支持平台  
_**INSERT [ AFTER | BEFORE ] output_section**_ 指定在预设linker script命令被执行之前或是之后加上或加入特定的输入section到输出section

符号赋值
----

### 简单赋值命令范例

可以在脚本中使用类C语言语法对符号进行赋值，例：
```
 symbol = expression ;
 symbol += expression ;
 symbol -= expression ;
 symbol *= expression ;
 symbol /= expression ;
 symbol <<= expression ; symbol >>= expression ;
 symbol &= expression ;
 symbol |= expression ;

一段简单的范例：

floating_point = 0;
SECTIONS
{
  .text :
    {
      *(.text)
      _etext = .;
    }
  _bdata = (. + 3) & ~ 3;
  .data : { *(.data) }
}
```
第一行：设定floating_point变量的链接地址为0

第七行：设置_etext的地址为.text之后

第九行：设置_bdata地址为_etext之后4字节alignment的位置。实则是对.data段的起始位置进行字节对齐

### HIDDEN

对输出ELF，把其中某个符号定义为目标文件内可见。对上文范例的symbol改写后：
```
HIDDEN(floating_point = 0);
SECTIONS
{
  .text :
    {
      *(.text)
      HIDDEN(_etext = .);
    }
  HIDDEN(_bdata = (. + 3) & ~ 3);
  .data : { *(.data) }
}
```
floating_point/_etext/_bdata 为ELF内可见，但是无法导出。即假设目标文件是库文件，其他文件也无法引用使用。

#### PROVIDE

链接器直接定义一个symbol，如果它没有在任何输入目标文件中定义，则链接时使用PROVIDE定义的符号。如果目标文件中有定义，则使用目标文件中的定义。例如：
```
SECTIONS
{
  .text :
    {
      *(.text)
      _etext = .;
      PROVIDE(etext = .);
    }
}
```
*   _**_etext**_不能在程序中定义，否则链接时有重复定义
*   _**etext**_则可定义在程序中定义，这样链接器会使用程序中的定义。

#### PROVIDE_HIDDEN

类似PROVIDE，差异只在于符号是HIDDEN

#### 源代码引用

C语言内可以引用链接器脚本中定义的变量。例如：
```C
 start_of_ROM   = .ROM;
 end_of_ROM     = .ROM + sizeof (.ROM);
 start_of_FLASH = .FLASH;
```
则C语言中可以如下引用：
```C
 extern char start_of_ROM, end_of_ROM, start_of_FLASH;
 memcpy (& start_of_FLASH, & start_of_ROM, & end_of_ROM - & start_of_ROM);
```
SECTION
-------

SECTION命令告诉链接器如何映射输入的段到输出段，且摆到内存的哪个位置。基本语法如下：
```
SECTIONS
{
  sections-command
  sections-command
  …
}
```
SECTION命令可能是以下命令的一种

*   ENTRY命令
*   符号赋值
*   输出段描述
*   覆盖描述

_**待补充。。。**_

参考文献
----

1.  <[GNU Link Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)>
2.  <[Linker Script初探 - GNU Linker Ld手冊略讀](https://wen00072.github.io/blog/2014/03/14/study-on-the-linker-script/)>