---
layout: post
title: 巧用Makefile给Go程序添加版本信息
date:   2018-08-19 22:30:00 +0800
category: Golang
tag: [golang]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/2018/golang-makefile.jpg
icon: code
---


* content
{:toc}

## 前言

Golang 的程序编译安装，如果善加使用 Makefile 文件，可以使开发过程更规范与高效，比如：

１、给编译的二进制文件“标记上”时间戳与源码版本号，方便程序更新与错误追溯。

２、开发时，监视文件修改，自动编译，执行测试用例，提高工作效率。

等等。。。


## 技巧列表

### 给二进制包标记时间戳与源码版本号

1、在 golang 代码中加入以下代码

```golang
package main

import (
	"fmt"
	"os"
)

// build info
var (
    BuildTime          = ""
    ProgramCommitID    = ""
	GoVersion          = ""
)

func init() {
	if len(os.Args) == 2 && (os.Args[1] == "-v" || os.Args[1] == "-version") {
		fmt.Println("go version: \t" + GoVersion) // \t 为 Tab 符号
		fmt.Println("Build Time: \t" + BuildTime)
		fmt.Println("Program Commit ID : \t" + ProgramCommitID)
		os.Exit(1)
	}
}
```

2、在 Makefile 执行编译的语句中加入常量：

```Makefile
# 默认在项目的根目录下执行 Makefile
GO_VERSION=$(shell go version)
BUILD_TIME=$(shell date +%F-%Z/%T)
ProgramCommitID=$(shell git rev-parse HEAD) # 项目是 Git 版本控制，则获取当前 commit id

LDFLAGS=-ldflags "-X 'main.GoVersion=${GO_VERSION}' -X main.BuildTime=${BUILD_TIME} -X main.ProgramCommitID=${ProgramCommitID}"

all:
	go build ${LDFLAGS} -v
```

３、以上代码加入任意项目中，成功编译后，比如可执行文件名为"executable"，这样执行 `executable -v`，会打印信息如下：

```
go version:     go version go1.11.1 linux/amd64
Build Time:     2018-10-29-CST/19:46:16
Program Commit ID :         53df91aeb41bef38530bdd5a5ebc4f334331a9bf
```

#### 原理解释

1、LDFLAGS是链接选项，`-X` 表示在编译时给变量赋值，通过 Makefile 中添加命令就可以灵活地给程序中的变量赋值。
2、golang 的 main 包的 init 函数是 golang 程序初始化阶段最后执行的一个 init 函数，上面代码的 init 函数执行完毕就 `os.Exit(1)` 推出，不会执行到 main 包的 main 主函数，这样，我们可以 `executable -v` 执行后查看程序包的信息后，关闭程序。

