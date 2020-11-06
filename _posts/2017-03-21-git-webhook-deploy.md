---
layout: post
title: Git+webhook+rsync项目自动部署的简单脚本
date:   2017-3-21 10:30:00 +0800
category: Git 
tag: [git]
---

* content
{:toc}
 

## 例子参考

- 本文的git webhook的server项目，开源了一个例子在github上，[git-webhook-deploy](https://github.com/lightfish-zhang/git-webhook-deploy)

## 前言

web开发者，开发到测试部署，在代码更新这一块如何做的呢。不要说原始的`compare + sftp`，大家常用的应该是使用版本控制工具，设置自动部署。    
本文讲述`svn`或`Git`版本控制 + `node.js`或`php`项目 + `rsync`从源码机同步代码到多台服务器，简单部署脚本的做法。以后或许会讲述`jenkins + docker + github(issue)`的全自动部署测试反馈的方案。    
　　
本文的目标，总结一句话，```提交代码到版本控制工具后自动执行工程构建\测试\部署```

## svn的简单做法

- svn的做法很简单，在机器上安装好svn, 运行命令`/usr/bin/svnserve -d -r /yourSvnDirectory`，
- 指定的svn存储文件夹`yourSvnDirectory`,结构如下

```
├── conf
├── db
├── format
├── hooks
├── locks
└── README.txt
```

- 其中`hooks`目录文件，可以设置svn的事件触发时执行的脚本，初始有以下文件

```
tree hooks/
hooks/
├── post-commit.tmpl
├── post-lock.tmpl
├── post-revprop-change.tmpl
├── post-unlock.tmpl
├── pre-commit.tmpl
├── pre-lock.tmpl
├── pre-revprop-change.tmpl
├── pre-unlock.tmpl
└── start-commit.tmpl
```

- 这些是示例文件，如`post-commit.tmpl`，是svn执行了提交事件后的执行脚本文件的模板，符合本文目标，我们将其名字改为`post-commit`，去掉`.tmpl`后缀，其内容修改为运维shell脚本，添加文件执行权限`chmod +x file`，就可以使用了。脚本内容下面再讲。

## 使用gitlab为版本控制，设置webhook

### 设置gitlab的事件通知的接口地址

- gitlab或github都提供事件通知功能，gitlab可以在项目设置的`Intergrations`选项里设置，如下图所示，可以设置接口地址与回调的事件类型

![brower-render-prase-html](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201702/gitlab-webhook-set.png)

- 接口通知格式可以参考官方说明[Webhooks Setting](https://git.yml360.com/help/web_hooks/web_hooks)
    + 请求头`request header`中包含通知的事件类型，如`X-Gitlab-Event: Push Hook`
    + 请求体`request body`中包含事件触发的项目的信息的json格式文本，如`requestBody.repository.name`是项目名

## 使用node.js作为接口webserver

- 这个不限于语言，仅仅作为事件接收与执行脚本web程序，笔者选择node.js，是因为node安装和配置方便
- webhook的server的执行逻辑如下(基于单进程的io多路复用的server)
    + 监听端口(与虚拟主机名)，接收到gitlab发起的http请求(发送事件类型与项目信息)，另外，获得内容后，可以直接响应200，继续执行下面操作
    + 根据事件类型与项目信息找到相关配置与脚本，然后，为了避免阻塞其他请求，fork出子进程去执行耗时的shell脚本，脚本内容下面再讲
    + 根据子进程返回的状态码与标准输出等信息，写入日志或者通知管理员(可接入IM工具)


## 运维脚本(拉代码/构建/测试/部署)

### 一个简单的消息通知脚本

- 运维人员需要知道部署过程发生了什么，所以可以在脚本中接入IM工具作为通知
- IM工具不限，笔者在工作中使用了钉钉，因为钉钉提供了机器人消息接口，而且工作的公司也使用了钉钉，简单脚本如下，定义一个shell函数方便使用，接口链接在钉钉的聊天设置中可以找到

```
robotMsg(){
    curl 'https://oapi.dingtalk.com/robot/send?access_token=xxxxx' \
    -H 'Content-Type: application/json' \
    -d "
    {\"msgtype\": \"text\",
    \"text\": {
        \"content\" : \"${1}\"
     }
    }"
}
```

### 拉取代码

- 拉取代码到服务器上，git工具的话，需要在gitlab设置一个服务器专用的账号，权限最好设置为只读，将服务器上的`id_rsa.pub`公钥添加到gitlab上，记得在服务器上提前将git服务的主机加入git来源可信任名单
- 拉取代码的shell脚本，少不了一些边界判断，与创建文件夹等操作

```sh
if [ ! -n "$PROJECT_NAME" ];then
    echo "please input project dir"
    exit
fi

if [ ! -n "$PROJECT_REPO" ];then
    echo "please input project repo"
    exit
fi

if [ ! -n "$PROJECT_BRANCH" ];then
    echo "please input project branch"
    exit
fi

if [ ! -x "$SRC_PATH" ];then
    mkdir $SRC_PATH -p
fi


if [ -x "$PROJECT_SRC" ];then
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} Start git pull ${PROJECT_REPO} ${PROJECT_NAME}")
	echo "Start git pull path:"$PROJECT_SRC
    cd $PROJECT_SRC
    echo "pulling source code..."
    git pull origin $PROJECT_BRANCH
    git clean -f
    git pull
else
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} Start git clone ${PROJECT_REPO} ${PROJECT_NAME}")
	echo "Start git clone path:"$PROJECT_SRC
	cd $SRC_PATH
	git clone $PROJECT_REPO $PROJECT_NAME
	cd $PROJECT_NAME
fi

git checkout $PROJECT_BRANCH
```


### 部署前端项目的脚本

- 现在主流的前端项目都是用`npm\webpack`作为打包发布工具，脚本可以这样写，注意的是，打包失败就不再往下执行，且部署到web目录下的时候，注意修改文件为web用户权限(像nginx严格限制web目录权限)

```sh
echo "build..."
npm install
if [ $? -eq 0 ];then
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm install success")
else
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm install fail")
    exit
fi
npm run build
if [ $? -eq 0 ];then
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm build success")
else
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm build fail")
    exit
fi
echo "release to web path"
if [ ! -x "$PROJECT_WEB" ];then
    mkdir $PROJECT_WEB -p
fi
yes|cp -a $PROJECT_SRC/dist/* $PROJECT_WEB
echo "changing permissions..."
chown -R $WEB_USER:$WEB_USERGROUP $WEB_PATH
echo "Finished."
```

- 顺便一提，如果前端工程的路由使用的是`http://domain/uri`格式的话，而不是`http://domain/#/uri`。对于Nginx来说，需要以下配置
- 前端工程为了使`uri`更加美化，使用类似静态资源的路径，如果从根目录进去是没问题，但是如果浏览器直接访问带uri的网址，会导致资源访问不到的情况，所以在访问文件不存在的情况，将资源重置为根目录下的index.html

```
        location / {
                try_files $uri /index.html;
        }
```

## node.js项目的部署

- 一般使用pm2来部署，pm2好像有git webhook组件可以使用，这里不提了，本文继续使用原来的方式介绍吧
- 这里的脚本也很简单，安装与启动或者重启，可能的话，加入`npm test`来测试也不错

```sh
echo "build..."
npm install
if [ $? -eq 0 ];then
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm install success")
else
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm install fail")
    exit
fi
export branch=$PROJECT_BRANCH
script="${PROJECT_SRC}/index.js"
count=`ps -ef |grep $script |grep -v "grep" |wc -l`
if [ 0 == $count ];then
    echo $script begin run
    pm2 start $script --name="${PROJECT_NAME} ${PROJECT_BRANCH}"
else
    echo $script restart
    pm2 restart $script --name="${PROJECT_NAME} ${PROJECT_BRANCH}"
fi
if [ $? -eq 0 ];then
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm start success")
else
    $(robotMsg "${PROJECT_NAME} ${PROJECT_BRANCH} npm start fail")
    exit
fi
echo "Finished."
```

### php项目的部署

- php简单，一般像`php-fpm`运行的项目，直接拉取代码就没事了，如果是php内建web服务，添加重启服务的脚本即可

## rsync从源码机同步代码到多台服务器

- 生产部署没那么简单了，高端的做法大概是`jenkins`+`docker`的`pipeline`做法，操作性好，不过本文还是讲述`脚本阶段`的做法
- 举一个有实践价值的命令，来讲述`rsync`工具的用法之一，本文不对`rsync`做更详细的介绍，因为网上资源够多了

```
rsync -vzau /www/* user@47.91.165.168:/www -e 'ssh -p 22' --exclude '.git'
```

- `v`选项，代表显示执行过程的详情
- `z`选项，`compress file data during the transfer`，传输时进行压缩
- `a`选项，封装备份模式，相当於 -rlptgoD，递回备份所有子目录下的目录与档案，保留链接档、档案的拥有者、群组、权限以及时间戳记。
- `c`选项，--checksum 打开校验开关，强制对文件传输进行校验
- `u`选项，--update 仅仅进行更新，也就是跳过所有已经存在于DST，并且文件时间晚于要备份的文件。(不覆盖更新的文件)

- --exclude=PATTERN 指定排除不需要传输的文件模式，本例子就对`.git`这些不必要的文件进行过滤，题外话，在Nginx配置中也应该对类似`.git`特殊信息的文件进行过滤
- -e, --rsh=COMMAND 指定使用rsh、ssh方式进行数据同步，一般是默认ssh的，本例子中命令故意写出，是因为有些机子的ssh端口号不一定是默认的22


## 环境变量修改参数

通过环境变量配置一些服务需要的参数，如数据库地址账号，其他服务的访问地址等，这个不详细说了
