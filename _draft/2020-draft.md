# 选举

- Redis 的哨兵选举与 cluster 选举的区别

哨兵

redis 实例启动时，可以选择 数据库模式 或 哨兵模式，一般 数据库 与 哨兵 是部署在不同的机器上。


- consul, etcd, zookeeper 相当于分布式服务架构中的哨兵


# 网络

- 网络模型中，Reactor 是同步非阻塞，proactor 是异步非阻塞

同步是指工作线程的调用是同步的，非阻塞是指io线程是非阻塞的，例如 golang的天然的编程方式

异步非阻塞，比较少见，给io事件传递函数数组

- 计算一个进程可以容纳多少个长连接

+ 内核的单个进程的文件描述符数量上限 `echo "655350" > /proc/sys/fs/file-max`
+ 临时端口范围
```conf
#  /etc/sysctl.conf
net.ipv4.ip_local_port_range = 10000 65535
```

+ IP_TABLE限制
+ TCP四元组，描述一个连接的数据结构的内存空间大小，如目标IP+目标端口，本进程设计的连接池的内存空间大小


# 灰度发布



