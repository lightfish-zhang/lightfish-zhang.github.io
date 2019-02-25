---
layout: post
title: 记一次使用mysql触发器的错误经历
date:   2017-4-25 10:30:00 +0800
category: Mysql 
tag: [mysql]
---

* content
{:toc}
 
## 前言

在工作中遇到一个需求，优惠券的记录的表，要求每生成一条优惠券记录，附带生成唯一的16个字符长度的号码，一开始想要mysql触发器实现，然而。。。

### 第一次尝试

#### 表的结构

```
create table `card`(
    `id` int(11) not null primary key,
    `code` char(16) default null,
    -- 省略其他字段
)ENGINE=InnoDB auto_increment=0 charset=utf8;

```

#### 新建触发器

```sql
use `database`;
delimiter $$
create trigger `card_insert_trigger` after insert
     on `card` for each row
     BEGIN
     update `card` set `code` = concat(left(concat(FROM_UNIXTIME(UNIX_TIMESTAMP(),'%Y%m%d'),floor(rand()*POWER(10,8))),16-length(`id`)),`id`)  where `code` is null or `code`='';
     END 
     $$
delimiter ;
```

#### 简单说明：

- `delimiter`表示修改mysql的sql语句的分界符，默认`;`，但是触发器中的语句往往带有`；`会对新建触发器的sql语句造成语法识别错误，于是改为不常用的字符，例如`$$`
- 这一条触发器的意思是，当插入新的一条数据后，执行`BEGIN ... END`中的语句，即根据id生成新的唯一的16个数字的字符串。

#### 执行的时候就遇到问题了：

```
insert into `card` (`id`, `expire_date`) values ('1', '2017-10-01 10:30:59') (error：Can't update table 'card' in stored function/trigger because it is already used by statement which invoked this stored function/trigger.)
```

#### 分析得知：

- INSERT可能会执行一些锁定，这可能会导致死锁。
- 从触发器更新表格将导致相同的触发器在无限递归循环中再次触发。
- 由上两条得知，mysql触发器不应该在对同一条记录的`insert`或`update`时又进行`insert`或`update`。

### 第二次尝试

#### 新建触发器

删除上一条失败的触发器，新建一条新的

```sql
use yml;
delimiter $$
create trigger card_insert_trigger before insert
on `card` for each row
BEGIN
set new.`code` = concat(left(concat(FROM_UNIXTIME(UNIX_TIMESTAMP(),'%Y%m%d'),floor(rand()*POWER(10,8))),16-length(new.`id`)),new.`id`);
END
$$
delimiter ;
```

简单说明:

- 在`insert`前，对记录进行修改，关键字是`new`，可以视作新数据的Object

#### 执行的时候就又遇到问题了：

- 插入新数据后，mysql没有报错，但是，认真观察数据就会发现，生成的`code`的最后几位并不是`id`，这就没办法保证`code`的唯一性　
- `id`是自增主键，只有mysql插入数据时，锁定表后才会生成的唯一`id`，而这次的触发器的操作是`before insert`

### 最终解决方案

因为上面两个触发器的尝试，一个是插入后触发对同一条数据的修改，会导致死锁或无限递归循环，另一个是在数据插入前没法获取唯一id来更新数据。似乎无路可走，只好放弃使用触发器，用最简单的办法吧，也就是在每一次`insert`语句后，立马执行以下，这样生成的`code`的操作不是原子性的，但是业务对此要求也不高，两条语句执行相差几毫秒的事儿

```sql
update `card` set `code`=concat(left(concat(FROM_UNIXTIME(UNIX_TIMESTAMP(),'%Y%m%d'),floor(rand()*POWER(10,8))),16-length(`id`)),`id`) where `code`='' or `code` is null;
```


## 参考文献

[触发器的语法说明](https://dev.mysql.com/doc/refman/5.7/en/trigger-syntax.html)