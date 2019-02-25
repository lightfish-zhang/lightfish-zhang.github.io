---
layout: post
title: nginx源码学习笔记-shell脚本安装配置
date:   2017-05-28 20:00:00 +0800
category: nginx 
tag: [shell]
---

* content
{:toc}
 

## 获取shell脚本的命令

### 使用option简单获取


```shell
for option
do
    echo option is $option
done
```

运行如下

```sh
# input
bash ./option.sh -a="123" -b='456' -c=789
# output
-a=123
-b=456
-c=789
```

### 获取=号后面的值

```sh
case "$option" in
    -*=*) value=`echo "$option" | sed -e 's/[-_a-zA-Z0-9]*=//'` ;;
        *) value="" ;;
esac
echo value is $value
```

### 指定参数值

以下shell片段，如果发现参数非脚本指定，就会报错

```sh
case "$option" in
    --help)                          help=yes                   ;;

    --prefix=)                       NGX_PREFIX="!"             ;;
    --prefix=*)                      NGX_PREFIX="$value"        ;;
    --sbin-path=*)                   NGX_SBIN_PATH="$value"     ;;
    --conf-path=*)                   NGX_CONF_PATH="$value"     ;;

    *)
        echo "$0: error: invalid option \"$option\""
        exit 1
    ;;
esac
```

- `case in ...; esac` 类似c语言的`switch case`用法
    + `;;`相当于`break`
    + 如`--sbin-path=*`的`*`是正则的通配符
    + `)` 是给条件与执行语句的划界限

### 感言

- shell脚本的参数，可使用`getopt`工具获取，可笔者觉得用法有点麻烦，认为自己写这样易读易懂一个简单版本比较好
- 该脚本是笔者研究nginx源码时看到nginx的configure的脚本获得启发