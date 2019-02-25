---
layout: post
title: 备忘录-gcc与gdb的使用方法与技巧
date:   2016-10-15 11:30:00 +0800
category: c/cpp 
tag: [编译]
---

* content
{:toc}

## 前言

gcc和gdb的使用方法与技巧，总结下当备忘录，以免想起使用再查找资料的麻烦。

## 关于gcc

### 如何看`man gcc`

在Linux环境下编程，使用c语言的感觉是得心应手的。不管是常用命令还是c语言的系统回调，gcc的使用方法只要`man gcc`命令即可查阅，但是刚看文档的新手可能看不太懂，我大概注解下，不定时更新。

- 开始`man gcc`，然后可见以下描述，gcc是GNU项目的c或者c++的编译器，大概有哪些命令

```
NAME
       gcc - GNU project C and C++ compiler

SYNOPSIS
       gcc [-c|-S|-E] [-std=standard]
           [-g] [-pg] [-Olevel]
           [-Wwarn...] [-Wpedantic]
           [-Idir...] [-Ldir...]
           [-Dmacro[=defn]...] [-Umacro]
           [-foption...] [-mmachine-option...]
           [-o outfile] [@file] infile...
```

gcc的命令详解可以往下看`DESCRIPTION`章节，我这里做中文的简单描述。

- 编译过程：预处理-编译-汇编-链接，可见以下`Makefile`文件

```Makefile
all:file.out

file1.i:file1.c
	gcc -E file1.c -o file1.i 

file2.i:file2.c
    cpp file2.c > file2.i

file1.s:file1.i
    gcc –S file1.i –o file1.s

file2.s:file2.i
    /usr/lib/gcc/x86_64-pc-linux-gnu/7.1.1/cc1 file2.i

file1.o:file1.i
    gcc -c file1.i -o file1.o

file2.o:file2.s
    as file2.s -o file2.o

file.out:
    gcc file1.o file2.o -o file.out

# or ld file1.o file2.o -o file.out

```

- 分步的编译的话，产生的文件的后缀名规范如下
    + `file.c`，`file.cc`，`file.cpp`是c/cpp的源文件，等待被预处理
    + `file.h` 头文件，也是源文件，用于声明函数，函数实质上是多个输入一个输出的代码块，编译时需要与`file.c`等源文件进行预处理，是因为编译器需要根据声明的类型，给输入值和输出值分配内存大小。常用于闭源或者已经编译好的`file.o`文件与头文件`file.h`一起使用。
    + `file.i` 已预处理，未编译
    + `file.s` 已编译，未汇编
    + `file.o` 已汇编，未链接
    + `file.out` 可执行文件


## 关于gdb

。。。待补


## 参考文献

- 《程序员的自我修养——链接、装载与库》
