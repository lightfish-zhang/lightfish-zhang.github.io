---
layout: post
title: 从最小的c程序复习编译原理
date:   2016-10-11 10:30:00 +0800
category: c/cpp 
tag: [编译, asm]
---

* content
{:toc}

## 前言
这是一篇读后感，从《程序员的自我修养》一书，复习了c语言的底层原理，下面以一个小例子来理解程序的编译链接


## 程序内容
不是print出HelloWorld字符串。这里最小程序的意义是，不引用任何头文件，直接打印出内容。因此使用了asm汇编指令。

以下代码用64位机器测试，asm指令中的寄存器`%rbx``%rcx``%rax`是`r`开头，如果是32位机器，将寄存器改为`e`开头即可。

~~~ c
/*
 * 64 bit
 */
char* str = "Hello world!\n";

void print()
{
    asm ( 
	    "movq $13,%%rdx \n\t"
	    "movq %0,%%rcx \n\t"
	    "movq $0,%%rbx \n\t"
	    "movq $4,%%rax \n\t"
	    "int $0x80 \n\t" ::"r" (str) : "rdx","rcx","rbx"
	);
}

void exit()
{
    asm(
	    "movq $42,%rbx \n\t"
	    "movq $1,%rax \n\t"
	    "int $0x80 \n\t"
       );
}

void nomain()
{
    print();
    exit();
}
~~~



附上Makefile

~~~ bash
minHelloWorld.out:minHelloWorld.o
	ld -static -e nomain -o minHelloWorld.out minHelloWorld.o
minHelloWorld.o:
	gcc -c -fno-builtin minHelloWorld.c
~~~

## 原理

### 编译参数说明

-  -fno-builtin 关闭GCC内置函数功能，如exit()，避免gcc优化替换
-  -static 使用静态链接，而不是默认的动态链接
-  -e xxx 指定程序的入口函数为xxx，使executable program的ELF文件头的Entry point address赋值为xxx函数的地址，使用readelf和objdump命令可证

~~~ bash
readelf -h minHelloWorld.out
...
Entry point address:               0x400127

objdump -d minHelloWorld.out
...
0000000000400127 <nomain>:
~~~

### 代码段(Sections)

~~~ bash
# 该命令可看详细的代码段、符号表等信息，详情见附录
 objdump -x minHelloWorld.out
~~~

- .text 保存程序指令
- .rodata 可见size为`0000000e`，等于"Hello world!\n"字符串大小，保存的就是它，而且只读(read only)。
- .data 保存的是str全局变量，可读写
- .comment 保存编译器和系统版本信息，对程序运行无用，可丢弃

~~~ bash
# strip 默认去除符号表、调试信息
strip minHelloWorld.out
strip --remove-section=.comment minHelloWorld.out
~~~


## 附录

### elf格式执行文件结构

~~~ bash
# objdump -x minHelloWorld.out

minHelloWorld.out:     file format elf64-x86-64
minHelloWorld.out
architecture: i386:x86-64, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0000000000400127

Program Header:
    LOAD off    0x0000000000000000 vaddr 0x0000000000400000 paddr 0x0000000000400000 align 2**21
         filesz 0x00000000000001c8 memsz 0x00000000000001c8 flags r-x
    LOAD off    0x00000000000001c8 vaddr 0x00000000006001c8 paddr 0x00000000006001c8 align 2**21
         filesz 0x0000000000000008 memsz 0x0000000000000008 flags rw-
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**3
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .rela.plt     00000000  00000000004000e8  00000000004000e8  000000e8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .plt          00000000  00000000004000e8  00000000004000e8  000000e8  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .text         00000059  00000000004000e8  00000000004000e8  000000e8  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  3 .rodata       0000000e  0000000000400141  0000000000400141  00000141  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .eh_frame     00000078  0000000000400150  0000000000400150  00000150  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .got.plt      00000000  00000000006001c8  00000000006001c8  000001c8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
  6 .data         00000008  00000000006001c8  00000000006001c8  000001c8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
  7 .comment      0000002d  0000000000000000  0000000000000000  000001d0  2**0
                  CONTENTS, READONLY
SYMBOL TABLE:
00000000004000e8 l    d  .rela.plt      0000000000000000 .rela.plt
00000000004000e8 l    d  .plt   0000000000000000 .plt
00000000004000e8 l    d  .text  0000000000000000 .text
0000000000400141 l    d  .rodata        0000000000000000 .rodata
0000000000400150 l    d  .eh_frame      0000000000000000 .eh_frame
00000000006001c8 l    d  .got.plt       0000000000000000 .got.plt
00000000006001c8 l    d  .data  0000000000000000 .data
0000000000000000 l    d  .comment       0000000000000000 .comment
0000000000000000 l    df *ABS*  0000000000000000 minHelloWorld.c
00000000004000e8 g     F .text  0000000000000029 print
0000000000400127 g     F .text  000000000000001a nomain
00000000006001d0 g       *ABS*  0000000000000000 __bss_start
00000000006001d0 g       *ABS*  0000000000000000 _edata
00000000006001d0 g       *ABS*  0000000000000000 _end
00000000006001c8 g     O .data  0000000000000008 str
0000000000400111 g     F .text  0000000000000016 exit

~~~
