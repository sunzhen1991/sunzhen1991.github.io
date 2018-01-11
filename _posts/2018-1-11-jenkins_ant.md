---
layout: post
title: "Jenkins+ant+doxygen"
categories: git
tags:  [jenkins ant doxgen]  
author: sunzhen
---

## jenkins安装

这两周我做了一些关于jenkins的工作，Jenkins，之前叫做Hudson，是基于Java开发的一种持续集成工具，简单来说就是通过各种插件把各种工作集成在一起构建。用于监控持续重复的工作，包括：

- 持续的软件版本发布/测试项目。
- 监控外部调用执行的工作。

### jenkins安装前准备

本文是把jenkins-x.xx.war包部署在tomcat下使用的，所以在安装jenkins之前还要先安装jdk和tomcat。这里要说一下环境变量的配置问题，若是在~/.bashrc下配置的环境变量可能到时候要在root用户下开启tomcat，在普通用户下或者使用sudo都不可以开启tomcat。这个我第一次配置ant时也显示了同样的问题。要么就是说“找不到JAVA_HOME或者JRE_HOME变量”要么就是说该命令所使用的JAVA_HOME目录是usr/lib/jvm/openjdk-xxx.xx什么的。目前还没有找到原因，不过我猜测试因为ubuntu自带的是openjdk所以有些地方可能我不小心安装时设置错了。

我的解决办法是：在etc/environment和~/.bashrc目录分别设置了java的路径配置，同时调用以下三个命令将java路径设置为系统的默认路径：

```shell
sudo update-alternatives  --install   /usr/bin/java   java     /usr/lib/jvm/jdk7/bin/java   300  
sudo update-alternatives  --install   /usr/bin/javac   javac    /usr/lib/jvm/jdk7/bin/javac   300  
sudo  update-alternatives  --install   / usr/bin/jar    jar         /usr/lib/jvm/jdk7/bin/jar  300  
```

### 下载安装jenkins

安装jenkins其实很简单，在官网上下载对应的jenkins-xxx.war包到电脑上，然后把它复制到tomcat目录下的webapps目录下就可以了。192.168.3.178是我的服务器地址，1234是tomcat监听端口。

## Doxygen

### 下载安装doxygen
Doxygen下载安装网上都有，对于如何调用doxygen网上也有，这些大致没有需要注意的。这是我使用doxygen后产生的页面，可以看到在INPUT目录中写了三个.cpp文件。

### 在jenkins下构建doxygen

在jenkins中安装插件 “HTML PUBLISHER PLUGIN”（使用Doxygen plugin会出现问题）。下载安装doxygen,生成相应配置文件，在build项中添加Execute shell，如： /Doxy_Dir/bin/doxygen  /path/to/doxyfile （相应的输入源文件和输出目录等设置已经都在doxyfile中设置好）在Post-build Actions中添加Publish HTML reports:填入doxyfile中设置好的文档输出目录。之后便能在任务成功build后在任务中看到附加上去的生成文档。

## Ant

### ant下载使用

Ant是用于自动开发的，下载ant也很简单，在终端输入sudo apt-get install ant即可。Ant的构建文件当开始一个新的项目时，首先应该编写Ant构建文件。构建文件定义了构建过程，并被团队开发中每个人使用。Ant构建文件默认命名为build.xml，也可以取其他的名字。只不过在运行的时候把这个命名当作参数传给Ant。构建文件可以放在任何的位置。一般做法是放在项目顶层目录中，这样可以保持项目的简洁和清晰。下面是一个典型的项目层次结构。 

- src存放文件。 
- class存放编译后的文件。 
- lib存放第三方JAR包。 
- dist存放打包，发布以后的代码。 

Ant构建文件是XML文件。每个构建文件定义一个唯一的项目(Project元素)。每个项目下可以定义很多目标(target元素)，这些目标之间可以有依赖关系。当执行这类目标时，需要执行他们所依赖的目标。每个目标中可以定义多个任务，目标中还定义了所要执行的任务序列。Ant在构建目标时必须调用所定义的任务。任务定义了Ant实际执行的命令。Ant中的任务可以为3类。 

- 核心任务。核心任务是Ant自带的任务。 
- 可选任务。可选任务实来自第三方的任务，因此需要一个附加的JAR文件。 
- 用户自定义的任务。用户自定义的任务实用户自己开发的任务。

下面用一个实验来讲解ant的使用。

使用Eclipse创建一个HelloWord的项目，项目放在/home/sun/mygit/helloworld/目录下。同时首先使用eclipse编写HelloWorld project：

```java
Package oata;  
Public class HelloWorld {  
  
Public static void main(String[] args){  
  
System.out.println(“Hello World”);  
  
}}  
```

在/home/sun/mygit/helloworld/目录下编写一个build.xml文件：

 ```xml
 <project name="HelloWord" basedir="." default="main">  
  
<property name="src.dir" value="src"/>  
  
<property name="build.dir" value="build"/>  
  
<property name="classes.dir" value="${build.dir}/classes"/>  
  
<property name="jar.dir" value="${build.dir}/jar"/>  
  
<property name="main-class" value="oata.HelloWorld"/>  
  
<target name="clean"> <delete dir="${build.dir}"/>  
  
</target> <target name="compile">  
  
<mkdir dir="${classes.dir}"/>  
  
<javac srcdir="${src.dir}" destdir="${classes.dir}"/>  
  
</target>  
  
<target name="jar" depends="compile">  
  
<mkdir dir="${jar.dir}"/>  
  
<jar destfile="${jar.dir}/${ant.project.name}.jar" basedir="${classes.dir}">  
  
<manifest> <attribute name="Main-Class" value="${main-class}"/>  
  
</manifest> </jar>  
  
</target>  
  
<target name="run" depends="jar">  
  
<java jar="${jar.dir}/${ant.project.name}.jar" fork="true"/>  
  
</target>  
  
<target name="clean-build" depends="clean,jar"/>  
  
<target name="main" depends="clean,run"/>  
  
</project> 
 ```
 然后打开终端，cd进入/home/sun/mygit/helloworld/目录，在该目录下输入antt命令。Ant就会自动编译产生一个build文件夹。在该文件夹下有.class和.jar等文件。

 ### 在jenkins下构建ant

 首先在/home/sun/mygit/helloworld/目录下执行git init命令创建.git文件夹（表示使用git版本库管理）。同时使用git add 命令把修改后的文件加入git中去。使用git commit命令提交。然后在源码管理中选择git，Git Repositories URL: /home/sun/mygit/helloworld在Build项中添加 Invoke Ant-Targets，并添加三行指令如下：lean compile jar  （即选择所要执行的build.xml中的target名）。之后执行便能看到build后的文件和Jar包。