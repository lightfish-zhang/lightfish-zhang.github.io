---
layout: post
title:  浅谈javascript引擎——js代码如何执行　
date:   2017-08-20 20:30:00 +0800
category: javascript
tag: [javascript]
---

## 前言

笔者被javascript的作用域搞得有点懵逼，干脆研究下js引擎，看代码是如何工作的

- 本文先列出一些知识点，比如编译，虚拟机的知识，不仅仅适于javascript


## 笔者的问题

笔者一开始就是出于对`javascript`的作用域感到疑惑而去研究引擎的，想不到这个坑越踩越深...

- 问题1，闭包的问题
    + 当代码的某一块作用域执行结束，该作用域内声明的变量应该要被回收，如果有的变量被其他作用域引用，那么这变量就需要等到其他作用域执行结束才能被回收
    + 闭包可以加以利用，在函数范式编程中发挥它的优势，另一方面，用不好就成内存泄露问题
    + 闭包详细说，相关术语有，引用，链式作用域，引用又有强引用与弱引用...

- 问题2，深复制，
    + 对于叶子节点都是基本类型的对象可以使用`let b=JSON.parse(JSON.stringify(a))`这个方法
    + 如果被复制的对象的叶子节点是”原生对象“，”function“，”Symbol“类型，那么该怎么深复制呢，可以用遍历的方法去检查各个节点的类型，是引用类型就判断是否原生对象或者function，是基本类型就判断是否”Symbol“这样的奇葩，以此判断是继续遍历还是直接引用还是赋值到目标对象的节点上

----
出于以上思考，笔者不禁对”js的对象如何实现“这个课题非常好奇


## 解释与编译的意思

### 形象的比喻

- 打个比喻，把需要被解释的代码比作地图的路线
- 使用解释的话，引擎就是一辆越野车，按照地图的路线去爬山涉野
- 使用编译的话，引擎就是火车与造铁轨的工人，先按照路线造出铁轨，再让火车在铁轨上跑起来

### 解释的过程

- 解释器的英文是`interpreter`
- 源代码对于解释器而言，像是命令一样，根据代码文本执行响应的命令，有代表性的例子是shell命令与term终端
    + term终端程序是一个二进制可执行文件，可以直接在操作系统上运行
    + term终端接收命令输入，解析“命令”这字符串，然后执行对应的命令

### 编译的过程

- 编译器的英文是`compiler`
- 源代码对于编译器而言，是原材料，编译器解析代码文本，然后生成可执行的二进制文件，可以直接在操作系统上运行
- 以Linux系统下C语言为例子，经过编译，生成如下`ELF`格式的二进制文件
    + C语言的编译步骤是: 预处理->编译->汇编->链接，最终生成二进制文件，
    + 上一点的编译是把预处理后的C代码翻译成汇编代码，汇编代码里没有了C语言那些`for`,`while`等语法，汇编语言有`if`条件判断,`jump`跳转，`move`修改寄存器数据,`add`累加等指令，变量会被分配内存地址用来存放数据，数组名/引用会被内存地址取代(本来就是地址的别名)

- 如下列出大概`ELF`文件格式，便于理解这个二进制文件为何可以在Linux操作系统上直接运行
    + `start address` 是程序执行的入口地址
    + `Program Header` 指出怎样创建进程映像，含有每个program header的入口，每个Program segment Header占 32-bytes(即e_phentsize大小)
    + `Sections`，包含section的信息，每个section header占 40-bytes (即e_shentsize大小)，这里不列出各个`sections`的名字与作用了

```
main.out:     file format elf64-x86-64
main.out
architecture: i386:x86-64, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0000000000400127

Program Header:
    LOAD off    0x0000000000000000 vaddr 0x0000000000400000 paddr 0x0000000000400000 

.......

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .rela.plt     00000000  00000000004000e8  00000000004000e8  000000e8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .plt          00000000  00000000004000e8  00000000004000e8  000000e8  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .text         00000059  00000000004000e8  00000000004000e8  000000e8  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE

....

```

### 不要纠结解释与编译的区别


> Python、Ruby、JavaScript都是“解释型语言”，是通过解释器来实现的。这么说其实很容易引起误解：语言一般只会定义其抽象语义，而不会强制性要求采用某种实现方式。例如说C一般被认为是“编译型语言”，但C的解释器也是存在的，例如Ch。同样，C++也有解释器版本的实现，例如Cint。

- 以`Java`为例，它被大众称为编译型语言，实际上它也有“解释执行“的特点，这点让人对概念产生疑惑
    + `Java`的源代码变成`bytecode`字节码，这一步被人称为"编译"
    + 接着，`bytecode`字节码（字节码）跑在`JVM`虚拟机上，VM会把输入的指令逐条直接执行，字节码是不能直接跑在操作系统上的，这一步可以称为"解释"

> VM并不是神奇的就能执行代码了，它也得采用某种方式去实现输入程序的语义，并且同样有几种选择：“编译”，例如微软的.NET中的CLR；“解释”，例如CPython、CRuby 1.9，许多老的JavaScript引擎等；也有介于两者之间的混合式，例如Sun的JVM，HotSpot。如果采用编译方式，VM会把输入的指令先转换为某种能被底下的系统直接执行的形式（一般就是native code），然后再执行之；如果采用解释方式，则VM会把输入的指令逐条直接执行。 

- 虚拟机有两个优化的方向
    + 解释，是把源程序中的指令逐条解释，不生成也不存下目标代码，后续执行没有多少可复用的信息
    + 稍微先进一点的解释器可能会优化输入的源程序，把满足某些模式的指令序列合并为“超级指令”；这么做就是朝着编译的方向推进

- 现在`javascript`的主流引擎，或者说大多数语言的虚拟机，都是在`编译`与`解释`之间取平衡，在给"优化代码工作的时间"与"运行效率"之间做取舍
- 所以，对于解释型语言与编译型语言的区分，不必纠结


## js引擎相关资料

### 相关术语

- 

## 以lv5做源码分析

### 关于lv5

- `lv5`的编写者`鈴木勇介`是一个日本人，在项目中到处都是炮姐里的角色名，`lv5`的名字怎么来，如果你也是废宅的程序员，不用多说，你知道的
- 为什么笔者用`lv5`做源码分析呢，因为这个js引擎实现相对简单，说白了笔者水平有限
- `lv5`实现了ECMA262 5th规范，使用C++语言编写

> iv is ECMA262 5.1 lexer and parser and engine project written in C++ / JS
> a lot of inspired from V8, SpiderMonkey, JavaScriptCore


### 对象的封装

#### 对象的类型

### 对象的内存布局

### 链式作用域

- 查找变量，当前作用域的变量字典里查找不到，就往上一层作用域查找

```cpp
// iv/lv5/railgun/scope.h  iv::lv5::railgun::CodeScope

  LookupInfo Lookup(Symbol sym) {
    const VariableMap::const_iterator it = map_.find(sym);
    if (it != map_.end()) {
      return it->second;
    } else {
      // not found in this scope
      if (scope_->direct_call_to_eval()) {
        // maybe new variable in this scope
        return LookupInfo::NewLookup();
      }
      return upper()->Lookup(sym);
    }
  }

```

> railgun，御坂美琴/超電磁砲，这里指`Stack VM`，或者说`Register VM` 

- 变量声明时，被存储在声明所在的作用域的变量字典`VariableMap`，lv5使用C++11的`unordered_map`

```cpp
  typedef std::unordered_map<Symbol, LookupInfo> VariableMap;
```

- 构建一个新的作用域时，`CodeScope`类的构建函数将成员`upper`设为上一层的作用域对象的指针，成员 `scope_nest_count_`设置为当前深度

```cpp

class VariableScope : private core::Noncopyable<VariableScope> {
 
 //...

 protected:
  // from catch or with
  explicit VariableScope(const std::shared_ptr<VariableScope>& upper)
    : upper_(upper),
      scope_nest_count_(upper->scope_nest_count() + 1) {
    // automatically count up nest
  }

  // from function
  VariableScope(const std::shared_ptr<VariableScope>& upper, uint32_t nest)
    : upper_(upper),
      scope_nest_count_(nest) {
  }

  // from eval or global
  VariableScope() : upper_(), scope_nest_count_(0) { }

 private:
  std::shared_ptr<VariableScope> upper_;
  uint32_t scope_nest_count_;
};

```

### 垃圾回收


// 未写完，待续...

## 参考资料

[javaScript引擎资料收集帖][1]
[lv5源码][2]

[1]:http://hllvm.group.iteye.com/group/topic/37596
[2]:https://github.com/Constellation/iv