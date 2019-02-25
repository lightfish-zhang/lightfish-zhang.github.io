---
layout: post
title: 浮点数运算问题和各种编程语言的解决方案
date:   2016-10-20 10:30:00 +0800
category: bug 
tag: [bug]
---

* content
{:toc}

## 前言
浮点运算的精确度，相信coder们都各自的体会，在各自常使用的语言中，都会不自觉陷入浮点运算的困扰，然后找到并有各自的解决方法．本文收集各种语言的解决方案，末尾放置浮点运算的科普和传送门
   
## 常见的语言作浮点运算

- c语言: `printf("%.17f\n", .1+.2)` 打印出 `0.30000000000000004`
- Java: `System.out.println(.1 + .2)` 打印出 `0.30000000000000004`，　`System.out.println(.1F + .2F)` 打印出 `0.3`
- Objective-C: `0.1 + 0.2;` 的结果是 `0.300000012`
- JavaScript: `console.log(.1 + .2)` 打印出`0.30000000000000004`
- PHP 有点意思， `echo .1 + .2` 打印出 `0.3`，PHP有这样一个的隐式转换的规律:

> PHP converts 0.30000000000000004 to a string and shortens it to “0.3”. To achieve the desired floating point result, adjust the precision ini setting: ini_set(“precision”, 17).

- 更多语言的浮点型运算例子请点传送门 [各种语言的浮点运算的结果比较]


## 解决方法

### c语言




## 浮点预算精度问题的探究

### 简单理解十进制与二进制的小数

#### 科学计数法表达

在十进制的5.25

```

   (5 * 10^0) + (2 * 10^-1) + (5 * 10^-2)
        5     +     2/10    +    5/100
   _________
   =  532.25

```

在十进制的5.25，也就是二进制的101.01

```

   (1 * 2^2) + (0 * 2^1) + (1 * 2^0) + (0 * 2^-1) + (1 * 2^-2)
       4     +      0    +     1     +      0     +    1/4
   _________
   =  5.25  Decimal

```

### 定点与浮点



### IEEE格式的数字以及运算过程

运算过程，相信科班毕业的coder对课本都有或多或少印象，从IEEE格式到cpu或者专门的浮点运算器的运算过程，电路门等，厚厚的课本 :)  
本文就不复制粘贴他人文章了，详细要点和传送门如下

- [浮点数的加减法运算]
- [IEEE Standards] IEEE标准官网



## 参考文献

- [各种语言的浮点运算的结果比较]
- [IEEE Standards]
- [Tutorial to Understand IEEE Floating-Point Errors]




[各种语言的浮点运算的结果比较]:http://0.30000000000000004.com/
[IEEE Standards]:http://standards.ieee.org/
[Tutorial to Understand IEEE Floating-Point Errors]:https://support.microsoft.com/en-us/help/42980/-complete-tutorial-to-understand-ieee-floating-point-errors
[浮点数的加减法运算]: http://share.onlinesjtu.com/mod/tab/view.php?id=184