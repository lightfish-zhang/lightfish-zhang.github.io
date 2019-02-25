---
layout: post
title: nodejs学习笔记——javascript和C++的编程
date:   2017-08-23 20:30:00 +0800
category: javascript
tag: [v8]
---


* content
{:toc}

## 前言

脚本语言与静态编译语言的结合，优点就不提了哈

## v8与C++的编程

参考如下文章

- [使用 Google V8 引擎开发可定制的应用程序](https://www.ibm.com/developerworks/cn/opensource/os-cn-v8engine/)


## node下的C++模块开发

### 规范写法

- node的C/C++模块使用`node-gyp`工具来编译
- 要看C++模块的开发示例，直接看node源码的`node/test/addons`的测试例子
- 最简单的一个示例如下

+ cpp源文件`binding.cc`, 其中的宏定义`NODE_SET_METHOD`与`NODE_MODULE`可见`node.h`

```cpp
#include <node.h>
#include <v8.h>

void Method(const v8::FunctionCallbackInfo<v8::Value>& args) {
  v8::Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(v8::String::NewFromUtf8(isolate, "world"));
}

// 模块的注册函数，也可以说是初始化函数
void init(v8::Local<v8::Object> exports) {

  // 注册方法，第一个参数是resv, 接受这个方法的对象，第二个参数是方法名，第三个是FunctionCallback, 这个方法的执行体
  NODE_SET_METHOD(exports, "hello", Method);
}

// 第一个参数是模块名，第二个参数是注册函数
NODE_MODULE(binding, init)
```

+ 编译需要的配置文件`binding.gyp`

```python
{
  'targets': [
    {
      'target_name': 'binding',
      'defines': [ 'V8_DEPRECATION_WARNINGS=1' ],
      'sources': [ 'binding.cc' ]
    }
  ]
}
```

+ 使用`node-gyp`编译以上两个文件
```sh
npm i -g node-gyp
node-gyp configure
node-gyp build
```

+ 以上步骤生成`binding.node`文件, 使用js代码测试

```js
const assert = require('assert');
const binding = require(`./build/Release/binding`);
assert.strictEqual(binding.hello(), 'world');
console.log('binding.hello() =', binding.hello());
```