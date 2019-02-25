---
layout: post
title: 关于Linux的/etc的的文件
date:   2016-11-20 16:20:00 +0800
category: Linux 
tag: [Linux]
---

* content
{:toc}
 
## 前言
Linux系统中，`/etc`是存放什么文件，其下的文件有什么作用

### /etc/passwd

每一行格式　　
用户名：密码：用户ID：主要组ID：GECOS：主目录：登录shell   

- 用户名：密码：用户ID：主要组ID：GECOS：主目录：登录shell
- 字段解释：
- 用户名：就是一个用户名，登录时候用的
- 密码：在旧的UNIX系统上，这个字段含有用户的加密密码，为了安全性，现在的linux均显示为x或*号
- 用户ID：linux内核用于识别用户的一个整数ID
- 主要组ID：linux内核用于识别用户主要组的一个整数ID
- GECOS：用户全名，安装linux时如果不输入全名，则显示为跟用户名一样，如果输入，则显示为全名（不可用于登录）
- 主目录：用户登录时，他的登录Shell将使用这个目录作为当前工作目录
- 登录Shell：用户登录时的默认Shell，在redhat 企业版中，登录shell通常是/bin/bash

#### example

````
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
lightfish:x:1000:1000:lightfish,,,:/home/lightfish:/usr/bin/zsh
````

如上所示，笔者`lightfish`使用`/usr/bin/zsh`作为shell，使用插件`oh-my-zsh`的界面和配色看起来舒服


### /etc/group

`/etc/passwd`的gid 对应着`/etc/group`文件的一条记录

每一行格式  
组名:口令:组标识号:组内用户列表  

- “组名”是用户组的名称，由字母或数字构成。与/etc/passwd中的登录名一样，组名不应重复。
- “口令”字段存放的是用户组加密后的口令字。一般Linux系统的用户组都没有口令，即这个字段一般为空，或者是*。
- “组标识号”与用户标识号类似，也是一个整数，被系统内部用来标识组。
- “组内用户列表”是属于这个组的所有用户的列表/b]，不同用户之间用逗号(,)分隔。这个用户组可能是用户的主组，也可能是附加组。

````
$ cat /etc/group
root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:syslog,lightfish
tty:x:5:
dialout:x:20:
lightfish:x:1000:
cdrom:x:24:lightfish
````

将用户分组是Linux系统中对用户进行管理及控制访问权限的一种手段。每个用户都属于某个用户组；一个组中可以有多个用户，一个用户也可以属于不同的组。当一个用户同时是多个组中的成员时，在/etc/passwd文件中记录的是用户所属的主组，也就是登录时所属的默认组，而其他组称为附加组。


### /etc/shadow

登录Linux时会要求输入用户名和密码。通常本地文件中会存储一份用户密码，并与用户输入对比，如果相同就允许用户登录。起初用户密码存储于/etc/passwd中，但由于/etc/passwd必须供所有用户读取，因此为避免密码破译，Unix系统将加密后的密码存储于/etc/shadow中，仅供超级用户可读（大型站点中使用NIS、LDAP、NIS+等方式存储）。   

/etc/shadow文件的每行由9个字段组成，以":"作为字段分隔符。每个字段的说明：

- 用户名（login name）。
- 加密后的密码（形如：$1$2eWq10AC$NaQqalCk3InEPBrIxjaJQ1）。如果密码是"*"或"!"，则表示这个不会用这个帐号来登录（通常是一些后台进程）。
- 密码最后修改时间，从1970年1月1日起计算的天数。
- 不可修改密码的天数。如果是0，表示随时可修改密码。如果是N，表示N天后才能修改密码。
- 密码可以维系的天数。如果设置为N，则表示N天后必须更新密码。设置为99999通常表示无需更新密码。
- 在密码必须修改前的N天，就开始提示用户需要修改密码。
- 密码过期的宽限时间。
- 帐号失效时间。也是UNIX时间戳格式。
- 最后一个字段是保留字段。

````
$ sudo cat /etc/shadow
root:!:17190:0:99999:7:::
daemon:*:17065:0:99999:7:::
bin:*:17065:0:99999:7:::
lightfish:$6$UKaqunyZ$hQOYL6Y4WdhzMU9hN2TM9QO9nFHqEwC4pYhr.R3lvaFa4d9mgQkN5G5.PGAs7/GIjykZVmV5eW.Cv7eAPDCxg0:17210:0:99999:7:::
````

#### 密码的格式

格式为`$id$salt$encrypted`

- id表示hash算法。起初密码用DES算法加密，但因随DES加密破解难度的降低，已用其他加密算法替代DES。在shadow文件中，密码字段如果以"$"打头，则表示非DES加密  

 | ID     | algorithm  |  
 |:---|:---|
 | $1$    | MD5        |
 | $2a$   | Blowfish   |
 | $5$    | SHA-256    |
 | $6$    | SHA-512    |

#### 密码的生成

未完持续....(代码过程　include <shadow.h> ....)

## reference

- [Shadow - 密码文件]
- [Linux Password & Shadow File Formats]

[Shadow - 密码文件]:http://www.berlinix.com/linux/shadow.php
[Linux Password & Shadow File Formats]:http://www.tldp.org/LDP/lame/LAME/linux-admin-made-easy/shadow-file-formats.html


