---
layout: post
title: 记录在Linux下理解Java项目的过程
date:   2018-02-24 20:30:00 +0800
category: java
tag: [java]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/2018/java-linux.png
---


* content
{:toc}


## 前言

- 因为工作需要，光鱼需要接手Java项目，之前是做golang, c/cpp开发的，刚查看的Java web项目的代码，对着xml配置文件、Annotation注解，maven的pom.xml配置等知识感到困惑，特别是对ide的使用方法
- 故此，写下一遍，我折腾之后的，回想起来的，在Linux下开发Java项目的理解，为什么强调Linux呢，我主张不依赖ide，在开发中多用命令行\shell脚本


## 写第一个项目

### 开发环境

- Linux Arch 4.14, Java openjdk 1.8.0, maven 3.5.2

### 设计模式

- Java==design pattern, 这句话深入人心，写Java好像没有人不强调设计模式的
- `abstract class`、`interface`、`inner class` 等面向对象的特性，完善了Java在设计模式上的探索，最经典的范例是GOF的23种设计模式
- 下面例子使用抽象工厂模式，最常见的模式。

#### 抽象工厂为例

- 本例子的代码的传送门是[抽象工厂例子1]，下面挑重点来讲
- Java的接口、抽象类等知识不在本文浪费篇幅，仅仅列出我认为该注意的地方
 
#### Java的工厂模式与xml映射配置

- 在本例中，`XMLUtil`负责从xml文件中读取类名与属性并实例化对象

```java

public class XMLUtil {
	//该方法用于从XML配置文件中提取具体类类名，并返回一个实例对象
	public static Object getBean(String tagName) {
		try {
			//创建文档对象
			DocumentBuilderFactory dFactory = DocumentBuilderFactory.newInstance();
			DocumentBuilder builder = dFactory.newDocumentBuilder();
			Document doc;
			doc = builder.parse(new File("config.xml"));
			NodeList nl = null;
			Node classNode = null;
			String cName = null;
			nl = doc.getElementsByTagName(tagName);
			classNode = nl.item(0).getFirstChild();

			cName = classNode.getNodeValue();
			//通过类名生成实例对象并将其返回
			Class c = Class.forName(cName);
			Object obj = c.newInstance();
			return obj;
		} catch (Exception e) {
			e.printStackTrace();
			return null;
		}
	}
}
```

```xml
<?xml version="1.0"?>
<config>
	<className>HaierFactory</className>
</config>
```

- 在main方法中，使用`XMLUtil.getBean()`获取对象然后执行方法，ps: `bean`是引用`Spring`的说法

```java

public class Main
{
    public static void main(String args[]){
        EFactory factory;
        Television tv;
        factory = (EFactory)XMLUtil.getBean();
        tv = factory.produceTelevision();
        tv.play();
    }
}
```

#### 命令行下执行Java程序

- 我在Linux下编写Java，使用vscode作为ide，仅仅使用它的语法提示功能。在编译与执行Java程序时，使用shell脚本。
- 以上面例子示范，执行`javac`编译，`-sourcepath`默认当前目录，如不是需要指定，`-d dir`指定输出目录

```s
mkdir build &&
javac -sourcepath src -d build src/Main.java &&
java -classpath build Main
```

- 一般来讲，源码目录与编译文件目录分来，最后执行`*.class`时指定class目录，并指定main函数的类名
- 在指定编译的文件`src/Main.java`时，`javac`遇到不认识的类型，会从`sourcepath`目录下查找，同时也编译相对应的`*.class`文件，但是如果是方法工厂模式下，往往是运行时才会通过一个类名来实例化一个对象，这就必须要提前手动编译好所需的`*.class`文件，可以使用下面命令

```shell
javac -sourcepath src -d build src/*.java
```

- 或者指定一个fileList文件，内容是java文件的路径，以空格或者换行符隔开

```
javac -sourcepath src -d build @fileList.txt
```

- 之后改进一下，添加`MANIFEST.MF`文件，打包成jar包，单文件容易管理，还能指定`Main Class`

```shell
#!/bin/bash
fileName=$(basename `pwd`).jar
if [ -d "build" ]; then
    rm build -rf
fi
mkdir build
javac -sourcepath src -d build src/*.java &&
jar cvfm ${fileName} MANIFEST.MF -C build . &&
echo -e "\n--------------------------------\n build success \n release ${fileName}\n--------------------------------\n"
rm build -rf &&
java -jar ${fileName} ${*}
```

- 需要额外注意的是，`MANIFEST.MF`文件最后要换行，否则最后一个属性不被java识别


### 使用maven规范化项目

- 上面的例子，代码与编译脚本都有点土，作为Java练习是够的，但是企业级开发下必须有所规范，现在最常用的管理工具是maven
- 接下来还是使用上面例子的代码，只是做规范下的改动，代码传送门[抽象工厂例子2]

#### 使用maven的archetype构建项目

- 有个小技巧，提前下载archetype-catalog.xml文件到本地

```s
curl http://repo.maven.apache.org/maven2/archetype-catalog.xml -o ~/.m2/repository/catalog.xml
```

- 在你的工作目录下执行快速创建的命令，当然使用本地下载好的archetype-catalog.xml，添加`-DarchetypeCatalog=local`，另外设置好自己项目的artifactId与groupId

```s
mvn -B archetype:generate \
  -DarchetypeGroupId=org.apache.maven.archetypes \
  -DgroupId=cn.lightfish.design_patterns \
  -DartifactId=design-patterns-java \
  -DarchetypeCatalog=local

```

- 项目执行完生成以下目录结构

```
design-patterns-java
├── pom.xml
├── src
│   ├── main
│   │   └── java
│   │       └── cn
│   │           └── lightfish
│   │               └── design_patterns
│   │                   └── App.java
│   └── test
│       └── java
│           └── cn
│               └── lightfish
│                   └── design_patterns
│                       └── AppTest.java
```

- maven遵循约定优于配置的原则，`src/main/java`存放源代码，`src/test/java`存放测试代码，强约定无可改
- 现在项目可以执行的命令是

```
mvn package
java -cp target/design-patterns-java-1.0-SNAPSHOT.jar cn.lightfish.design_patterns.App
```

- 显示hello world，在这之前mvn也执行了compile与test，maven的更多知识请看[maven-document]


#### 让上面的抽象工厂例子符合maven项目规范

- 代码传送门[抽象工厂例子2]，目录结构如下，代码的改动很少，添加声明`package`与留意`import`即可

```
src/main/java/cn/lightfish/design_patterns/creational/simple_factory
├── Administrator.java
├── Employee.java
├── Manager.java
├── UserDAO.java
├── UserFactory.java
└── User.java
src/test/java/cn/lightfish/design_patterns/creational/simple_factory
└── SimpleFactoryTest.java
```

- 进行测试，设置参数`-Dtest`为指定测试类的名字，这样可以针对单一测试类进行测试，另外该参数还支持正则表达式进行一个组的测试，也可在pom.xml内配置

```s
mvn -Dtest=SimpleFactoryTest test
```

- 上例使用`junit 4.8.1`，`junit`的`3.*`与`4.*`的用法有些区别


### 关于注解 Annotation

- 光鱼一开始的理解，是将java的Annotation类比c/cpp的预编译指令或者宏定义，后来却发现Annotation的机制很灵活，可以自定义编写自己的Annotation


## Reference

- [抽象工厂例子1]
- [抽象工厂例子2]
- [maven-document]


[抽象工厂例子1]:https://github.com/lightfish-zhang/design-pattern-java/tree/master/03-abstract-factory
[抽象工厂例子2]:https://github.com/lightfish-zhang/design-patterns-java/tree/master/src/main/java/cn/lightfish/design_patterns/creational/simple_factory
[maven-document]:http://maven.apache.org/guides/getting-started/index.html#How_do_I_make_my_first_Maven_project