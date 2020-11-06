---
layout: post
title: 如何定制Nginx配置
date:   2018-08-30 20:30:00 +0800
category: nginx
tag: [server]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201901/nginx-icon.png
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/2018/nginx.png
icon: note
---


* content
{:toc}


## 前言

nginx 配置的备忘录

首先参考 Nginx 官方的 Full Example config: 

- [Full Example 1](https://www.nginx.com/resources/wiki/start/topics/examples/full/)，基础的配置
- [Full Example 2](https://www.nginx.com/resources/wiki/start/topics/examples/fullexample2/)，在前者的基础上优化性能，如 gzip, tcp_nodelay, send_lowat 等

在官方给出的 full example 配置建议上，如何定制符合自身的业务场景的配置，让我们看一下选项

## HTTP 负载限制

场景：

- 长连接的流服务限制同一IP下的连接数
- HTTP服务限制 DDos 攻击
- 后端服务负载能力有限，须限制流量防止宕机

注意：限制性的配置会限制自身服务的吞吐量，在流量高峰期，超出服务的设计限制，会引起客户端访问服务失败的问题，需要做好评估


### 限制速度

模块： ngx_http_core_module

配置项： limit_rate, limit_rate_after

默认值与作用上下文：[官方文档](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate)

摘抄官方解释：

限制向客户端传输的响应速率，速率以每秒字节数指定，零值禁用速率限制，根据请求设置限制，因此如果客户端同时打开两个连接，则总速率将是指定限制的两倍。

Example:

```
location /flv/ {
    flv;
    limit_rate_after 500k;
    limit_rate       50k;
}
```

可以配合 `if` ，使用变量

```
if ($slow) {
    set $limit_rate 4k;
}
```

### 限制同一时刻保持的HTTP请求数

模块: ngx_http_limit_conn_module

配置项: limit_conn, limit_conn_zone, limit_conn_log_level, limit_conn_status

默认值与作用上下文：[官方文档](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html)

Example:

```
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
    limit_conn_log_level error;
    limit_conn_status 503;
}
```

说明：

- 设置辨别空间的变量，视需求采用不同的变量，例如
    + `$binary_remote_addr`，客户端ip地址，这是 `$remote_addr` 变量的二进制格式，比如 `$remote_addr` 字符串占据7~15字符，在64位系统中占据64 bytes内存，而二进制格式占据4 bytes内存，了解这个对于缓存分配有指导作用
    + `$server_name`，域名
- 设置空间名字 `zone=zoneName`
- 设置zone的共享内存大小`10m`，以及连接数的最大允许值。当连接数超过设置的限制、或者存储空间耗尽的时候，服务器将返回503状态值
    ＋ 对于 `$binary_remote_addr` 为key，键值对记录大概64 bytes，设置 1M 可以保存大约1万6千个64字节的记录
- 设置同一时间同一个"空间"的连接数 `limit_conn zoneName 1`
- 设置发生超出连接限制值时，记录的日志等级与返回状态码

具体场景：

- 长连接的流式服务，可以限制同一IP连接数，使用 `$binary_remote_addr` 作为空间变量
- 保护HTTP后端服务，免于高峰流量的宕机，使用 `$server_name` 作为空间变量

### 限制时间间隔内新建HTTP请求数

模块：ngx_http_limit_req_module

配置项：limit_req, limit_req_log_level, limit_req_status, limit_req_zone

默认值与作用上下文：[官方文档](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)

Example:

```
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;
limit_req_zone $binary_remote_addr zone=perip2:10m rate=60r/m;
limit_req_zone $server_name zone=perserver:10m rate=10r/s;

server {
    ...
    limit_req zone=perip burst=5 nodelay;
    limit_req zone=perserver burst=10;
    limit_req_status 503;
}
```

说明：

- 用法与上面的 ngx_http_limit_conn_module 类似，区别在于这里是限制时间间隔内新建TCP连接数
- 这个模块是平滑的限流，即时比如 10r/s 的rate, 当同一时刻来了10个请求，nginx会固定以 100 ms 的时间间隔处理这些请求
- limit_req 的 burst 参数设置等待的队列，比如设置 burst = 10, 如果同一时刻来了 11 个请求，nginx会将 10 个请求放进队列中，丢弃1个请求，返回503
- 平滑的限流不太实际，可以设置nodelay来不再延时请求，如 10r/s，不再限制 100ms 处理一个请求，而是 1s 内最多处理 10 个请求



## HTTP 的负载限制

场景：

- 根据业务场景定制相应的 http header/body 的缓存大小，优化服务器的资源利用效率，过滤非法请求

### 限制 HTTP 的 header/body 缓存大小

模块: ngx_http_core_module

配置项: client_header_buffer_size, large_client_header_buffers, client_max_body_size, client_body_buffer_size

命令详情：[官方文档](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size)

Example:

```
http {
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;
  client_max_body_size 1024m;
  client_body_buffer_size 10m;
}
```

## HTTP 静态资源的压缩

模块: ngx_http_gzip_static_module

命令详情：[官方文档](http://nginx.org/en/docs/http/ngx_http_gzip_static_module.html)

Example:

```
# 开启gzip
gzip on;

# 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
gzip_min_length 1k;

# gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
gzip_comp_level 2;

# 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;

# 是否在http header中添加Vary: Accept-Encoding，建议开启
gzip_vary on;

# 禁用IE 6 gzip
gzip_disable "MSIE [1-6]\.";
```


## 区别对待IP地址

模块: ngx_http_geo_module

命令详情：[官方文档](http://nginx.org/en/docs/http/ngx_http_geo_module.html)

Example 1, 设置白名单，且配合其他模块，对名单内ip进行限制访问、连接数、请求速率等

```
 geo $whiteipflag  {
  default 1;
  127.0.0.1 0;
  192.0.0.0/8 0;
  103.20.102.0/24 0;
 }

 map $whiteipflag $limitvar {
  1 $binary_remote_addr;
  0 "";
 }

 limit_conn_zone $limitvar zone=limitip:10m;
 limit_req_zone $limitvar zone=perip:10m rate=1r/s;
   
 server {
        listen       80;
        server_name  localhost;
   
        location ^~ /download/ {
                if ($whiteipflag = 1){
                  return 403;
                }
                if ($whiteipflag = 0) {
                  set $limit_rate 500k;
                }
                limit_conn limitip 4;
                limit_req zone=perip burst=5 nodelay;
        }
 }
}
```


Example 2, 区分地域，代理转发不同的服务器

```
geo $country {
  default default;
  include conf/geo.conf;
  delete 127.0.0.0/16;
  proxy 192.168.100.0/24;
  proxy 2001:0db8::/32;
  
  127.0.0.0/24 uk;
  127.0.0.1/32 us;
}

upstream  uk.server {
  server 127.0.0.1:9090;
}

upstream  us.server {
  server 127.0.0.2:9090;
}

upstream  default.server {
  server 127.0.0.3:9090;
}

server {
  location / {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://$geo.server;
  }
}
```


摘要常用的选项的说明：

1）delete：删除指定的网络
2）default：如果客户端地址不能匹配任意一个定义的地址，nginx将使用此值。 如果使用CIDR，可以用"0.0.0.0/0"代替default。
3）include： 包含一个定义地址和值的文件，可以包含多个。
4）proxy：定义可信地址。 如果请求来自可信地址，nginx将使用其“X-Forwarded-For”头来获得地址。 相对于普通地址，可信地址是顺序检测的。
5）proxy_recursive：开启递归查找地址。 如果关闭递归查找，在客户端地址与某个可信地址匹配时，nginx将使用"X-Forwarded-For"中的最后一个地址来代替原始客户端地址。如果开启递归查找，在客户端地址与某个可信地址匹配时，nginx将使用"X-Forwarded-For"中最后一个与所有可信地址都不匹配的地址来代替原始客户端地址。
6）ranges：使用以地址段的形式定义地址，这个参数必须放在首位。为了加速装载地址库，地址应按升序定义。

## nginx 日志压缩

纯文本日志过大，可以使用gzip进行压缩，可以设置类似以下脚本，注意给 nginx 发送进程信号USER1，打开新的log文件

```
newLogFile=access.`date +%s`.log
mv access.log $newLogFile
nginx -s reopen
gzip -d $newLogFile.gz $newLogFile
rm $newLogFile
```


## Nginx 官方的 Full Example

很全面的配置，有复杂的代理选项、针对性能优化的配置

### 额外注意的，有可能引发问题的配置

- worker_rlimit_nofile worker进程的最大打开文件数限制，如果没设置的话，这个值为操作系统的限制。设置后你的操作系统和Nginx可以处理比“ulimit -a”更多的文件，所以把这个值设高，这样nginx就不会有“too many open files”问题了

- worker_connections 定义 Nginx 每个进程的最大连接数，默认是 1024，按现在的服务器性能进化，这个值远低于实际需求。

- multi_accept 默认情况下，Nginx 进程只会在一个时刻接收一个新的连接，我们可以配置multi_accept 为 on，实现在一个时刻内可以接收多个新的连接，提高处理效率。该参数默认是 off，建议开启。

- server_names_hash_max_size，使用CPU高速缓存储存的域名散列表的缓存大小，如果 nginx 内定义过多的域名，需要调整该参数，以空间换时间，减少查询操作的复杂度。同时需要调整 server_names_hash_bucket_size 选项。

- sendfile，对应 Linux 2.6.33 之后出现的系统调用 `sendfile`，减少用户态与内核态的上下文切换，直接在内核态完成缓存的复制，建议开启。


### Nginx自动优化项，用户不必太过注意

- use 选择IO多路复用的轮询方式，例如 epoll，缺省时，Nginx会根据系统自动选择。

- worker_cpu_affinity　配置 Nginx 进程与 CPU 亲和力的参数，缺省时，将由 Nginx 按需自动分配。



nginx.conf

```
user  www www;
worker_processes  2;
worker_cpu_affinity auto;
pid /var/run/nginx.pid;

# [ debug | info | notice | warn | error | crit ]
error_log  /var/log/nginx.error_log  info;

events {
  worker_connections   2000;
  # use [ kqueue | rtsig | epoll | /dev/poll | select | poll ] ;
  use epoll;
  multi_accept on;
}

http {
  include       conf/mime.types;
  default_type  application/octet-stream;

  log_format main      '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$gzip_ratio"';

  log_format download  '$remote_addr - $remote_user [$time_local]  '
    '"$request" $status $bytes_sent '
    '"$http_referer" "$http_user_agent" '
    '"$http_range" "$sent_http_content_range"';

  client_header_timeout  3m;
  client_body_timeout    3m;
  send_timeout           3m;

  client_header_buffer_size    32k;
  large_client_header_buffers  4 32k;

  gzip on;
  gzip_min_length  1100;
  gzip_buffers     4 8k;
  gzip_types       text/plain;

  output_buffers   1 32k;
  postpone_output  1460;

  sendfile         on;
  tcp_nopush       on;

  tcp_nodelay      on;
  send_lowat       12000;

  keepalive_timeout  75 20;

  # lingering_time     30;
  # lingering_timeout  10;
  # reset_timedout_connection  on;


  server {
    listen        one.example.com;
    server_name   one.example.com  www.one.example.com;

    access_log   /var/log/nginx.access_log  main;

    location / {
      proxy_pass         http://127.0.0.1/;
      proxy_redirect     off;

      proxy_set_header   Host             $host;
      proxy_set_header   X-Real-IP        $remote_addr;
      # proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;

      client_max_body_size       10m;
      client_body_buffer_size    128k;

      client_body_temp_path      /var/nginx/client_body_temp;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_send_lowat           12000;

      proxy_buffer_size          4k;
      proxy_buffers              4 32k;
      proxy_busy_buffers_size    64k;
      proxy_temp_file_write_size 64k;

      proxy_temp_path            /var/nginx/proxy_temp;

      charset  koi8-r;
    }

    error_page  404  /404.html;

    location /404.html {
      root  /spool/www;

      charset         on;
      source_charset  koi8-r;
    }

    location /old_stuff/ {
      rewrite   ^/old_stuff/(.*)$  /new_stuff/$1  permanent;
    }

    location /download/ {
      valid_referers  none  blocked  server_names  *.example.com;

      if ($invalid_referer) {
        #rewrite   ^/   http://www.example.com/;
        return   403;
      }

      # rewrite_log  on;
      # rewrite /download/*/mp3/*.any_ext to /download/*/mp3/*.mp3
      rewrite ^/(download/.*)/mp3/(.*)\..*$ /$1/mp3/$2.mp3 break;

      root         /spool/www;
      # autoindex    on;
      access_log   /var/log/nginx-download.access_log  download;
    }

    location ~* ^.+\.(jpg|jpeg|gif)$ {
      root         /spool/www;
      access_log   off;
      expires      30d;
    }
  }
}
```

## 附录

### CIDR值，参考表

[在线的子网掩码计算器](http://tool.chinaz.com/tools/subnetmask)

```
子网掩码                 CIDR值
255.0.0.0                 /8
255.128.0.0               /9
255.192.0.0               /10
255.224.0.0               /11
255.240.0.0               /12
255.248.0.0               /13
255.252.0.0               /14
255.254.0.0               /15
255.255.0.0               /16
255.255.128.0             /17
255.255.192.0             /18
255.255.224.0             /19
255.255.240.0             /20
255.255.248.0             /21
255.255.252.0             /22
255.255.254.0             /23
255.255.255.0             /24
255.255.255.128           /25
255.255.255.192           /26
255.255.255.224           /27
255.255.255.240           /28
255.255.255.248           /29
255.255.255.252           /30
```