---
layout: post
title: java开发-备忘录
date:   2017-11-30 19:30:00 +0800
category: java
tag: [java]
---


* content
{:toc}

## 前言

光鱼的常写的语言是c/cpp, golang, javascript，代码风格一向简洁直接，功能适当封装不过度。工作中又不可避免使用java这种套路特别多的语言，它约定俗成的东西太多了，故做备忘录

## 关于注解Annotation

### 注解的介绍

- 注解在java代码里通常使用`@`开头，给编译器看的，类似c语言的预处理指令，生成更多的代码，减少人力重复工作，java的注解具有更多的功能 
- 比如spring boot使用的`@RestController`给一个类注解成一个http rest控制器，`@RequestMapping("/hello")`注解一个方法映射到http uri的回调处理

### 一些注解的笔记

#### 常用的

- `@Deprecated`在其他语言也很常见，多用于注释，表示不赞成再使用某方法 
- `@Override`子类覆盖父类的方法时建议使用


#### 不常用的

- `@SuppressWarnings("serial")`, Suppress屏蔽，serial是序列化的意思，屏蔽序列化的警告。在实现Serializable接口的时候是需要添加serialVersionUID属性的，但是不需要序列化且不想添加这个属性，就使用该注解屏蔽

## 特别的技巧


## 阅读xml

第一次接触java工程代码，光鱼刚开始看一堆xml文件时是蒙圈的，整理下思路

### mvn与pom.xml

- mvn简单理解为java的包管理器，在企业团队开发时，最好使用局域网的服务器下载依赖包，保证来源包的统一安全又能节省带宽，配置在安装目录下`maven/conf/settings.xml`
- pom.xml配置了依赖包、java版本等信息，常用的标签有
    + `<dependency>` 依赖包信息 
    + `<build><finalName>` 打包jar的名字格式 
    + `<parent>`父pom配置
    + `<project.jdk>` 指定java sdk的版本

- 像shell一样，在xml里面可以使用`${abc}`这样的变量

### spring的bean与xml


> Bean的中文含义是“豆子”，顾名思义JavaBean是一段Java小程序。JavaBean实际上是指一种特殊的Java类，它通常用来实现一些比较常用的简单功能，并可以很容易的被重用或者是插入其他应用程序中去。所有遵循一定编程原则的Java类都可以被称作JavaBean。

> Java Bean是基于Java的组件模型，由属性、方法和事件3部分组成。在该模型中，JavaBean可以被修改或与其他组件结合以生成新组件或完整的程序。

#### spring的ioc容器(控制反转 “Inversion of Control”)

- Spring设计的核心是org.springframework.beans包，它的设计目标是与JavaBean 组件一起使用。
- 下一个最高级抽象是BeanFactory接口，它是工厂设计模式的实现，允许通过名称创建和检索对象。BeanFactory 也可以管理对象之间的关系。
- bean 工厂的概念是 Spring 作为 IOC 容器的基础。IOC 将处理事情的责任从应用程序代码转移到框架。正如我将在下一个示例中演示的那样，Spring 框架使用 JavaBean 属性和配置数据来指出必须设置的依赖关系。
- Bean定义的方式有：XML文件、注解Annotation、Java Code、用户自定义（？）


### 用法示例

- 使用`settings.properties`文件，`annotation`注解、`spring bean`的xml配置，可以构建工厂模式下一个类的方法、属性、事件绑定等的设置。

#### 数据库配置的例子

- `settings.properties`

```
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://0.0.0.0/yourDB
jdbc.username=xxx
jdbc.password=xxx
```

- `spring bean`的xml配置
    + 以下的`systemProperties`是系统属性，使用Java命令行`-D`可指定，如`java -Dmyproject.configuration.filepath=file://$HOME/config/settings.properties`

```xml
	<util:properties id="settings" location="#{systemProperties['myproject.configuration.filepath']?:'classpath:settings.sample.properties'}">
	</util:properties>

	<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close">
		<property name="driverClassName" value="#{settings['jdbc.driverClassName']}" />
		<property name="url" value="#{settings['jdbc.url']}" />
		<property name="username" value="#{settings['jdbc.username']}" />
		<property name="password" value="#{settings['jdbc.password']}" />		
		<property name="maxActive" value="#{settings['pool.maxActive']}" />
		<property name="maxIdle" value="#{settings['pool.maxIdle']}" />
		<property name="minIdle" value="#{settings['pool.minIdle']}" />
		<property name="defaultAutoCommit" value="#{settings['pool.defaultAutoCommit']}" />
		<property name="testOnBorrow" value="true" />
		<property name="validationQuery" value="SELECT 1" />
	</bean>
```

- Java code

```java
package org.apache.commons.dbcp;

public class BasicDataSource implements DataSource {
    protected volatile boolean defaultAutoCommit = true;
    protected transient Boolean defaultReadOnly = null;
    protected volatile int defaultTransactionIsolation = -1;
    protected volatile String defaultCatalog = null;
    protected String driverClassName = null;
    protected ClassLoader driverClassLoader = null;
    protected int maxActive = 8;
    protected int maxIdle = 8;
    protected int minIdle = 0;
    protected int initialSize = 0;
    protected long maxWait = -1L;

    //...
}

```

- 如上，配置完成