---
layout: post
title: "gerrit replication"
categories: git
tags:  [gerrit]  
author: sunzhen
---

gerrit replication用于完成gerrit主从库之间的代码库同步。

## Gerrit服务器配置

gerrit服务器A（主）：

- 操作系统： ubuntu
- 地址：192.168.3.178
- Http访问：192.168.3.178:8087
- Gerrit认证方式：development_become_any_account这是http访问gerrit服务器的认证方式其他的还有openid等方式。
- ssh端口：29418
- 注意在选择是否下载replication pligin选项的时候默认是不下载的，这时候要在选项后面写为y才会下载。

Gerrit服务器B（从）：

- 操作系统： ubuntu
- 地址：192.168.4.64
- Http访问：192.168.3.178:8087
- Gerrit认证方式：development_become_any_account
- ssh端口：29418
-通过ssh的29418端口访问B：ssh -p 29418 xxxxx@192.168.4.64
- 为了简单起见我的两个管理员账户都是xxxxx。

这时候gerrit开始后，在主机A上访问http://192.168.4.64:8087。然后加入自己的公钥。这个是为了把A的公钥加入到B中，通过port 29418访问B。

## replicattion.config

在A中创建repo1，并且复制到B中对应位置。

在gerrit目录下的etc目录中新建一个replication.config。这样你重新加载replication插件时就会读取该文件:

```shell
[remote "192.168.4.64"]  
  
  url = ssh://xxxxx@192.168.4.64:29418/${name}.git  
  
  push = +refs/heads/*:refs/heads/*  
  
  push = +refs/tags/*:refs/tags/*  
```
做完之后在A和B都开启的情况下。在主机A下面输入:

```shell
aaa@aaa-OptiPlex-7010:~$ ssh -p 29418 xxxxx@localhost gerrit plugin reload replication  
  
aaa@aaa-OptiPlex-7010:~$ ssh -p 29418 xxxxx@localhost replication start  --all  
```
这两个指令分别重载replacation配置和开始复制。时候查看A下面的gerrit目录下面的error_log文件。可以看到如下信息：

```shell
[2014-09-29 15:04:34,327] INFO  com.google.gerrit.server.plugins.PluginLoader : Reloading plugin replication

[2014-09-29 15:04:34,374] INFO  com.google.gerrit.server.plugins.PluginLoader : Unloading plugin replication

[2014-09-29 15:05:34,653] INFO  com.google.gerrit.server.plugins.PluginLoader : Cleaned plugin plugin_replication_140929_1458_5386789366483874484.jar


[2014-09-29 15:11:58,633] INFO  com.googlesource.gerrit.plugins.replication.ReplicationQueue : Push to ssh://xxxxx@192.168.4.64:29418/repo1.git references: [RemoteRefUpdate[remoteName=refs/heads/master, NOT_ATTEMPTED, (null)...fdb7a98b21c53545977e01e2c83a0c6a9359d160, srcRef=refs/heads/master, forceUpdate, message=null]]
```


## gerrit代码库实现push和pull的目标位不同主机

现在我想做的是当上传代码时代码不能上传到B主机，必须上传到A中，然后代码的pull操作是从B主机pull下来的。这时候要分为两步：

一：修改B主机的gerrit权限，使B中gerrit库除了管理员之外只能下拉代码而不能上传代码

设置完成后，当你在B（192.168.4.64）下使用非管理员账户执行git push语句时，会提示你错误信息。

二：编写一个config.sh脚本，该脚本有两个参数push和pull，该脚本大致的作用是：当使用./config.sh push命令时，该脚本自动修改参数，使git push的对象指向主机A（192.168.3.178），当使用./config.sh pull命令时，该脚本则修改相关参数使得该pull对象指向主机B。这就实现了从B下载代码，从A上传代码，然后A中代码库的改变又复制到B。

config.sh代码如下：
```shell
#!/bin/bash

#
# Purpose: git push to master and pull from mirror`s repository
# Date:    2014-10-9
# Author:  
#
if [ "$1" == "push" ]
then 
fullname_old=`git config --get user.name`
fullname=
read -p "Your Full Name In 192.168.3.178:8087[$fullname_old?]: " fullname
if [ -z "$fullname" ]; then
    fullname="$fullname_old"
fi
git config --global user.name "$fullname"

echo

sed -i '7d' .git/config
sed '7 iurl = ssh://'$fullname'@192.168.3.178:29418/repo1' -i .git/config

email_old=`git config --get user.email`
read -p "Your Email  In 192.168.3.178:8087[$email_old?]: " email
if [ -z "$email" ]; then
    email="$email_old"
fi
git config --global user.email "$email"
echo

git config --global color.diff auto
git config --global color.status auto
git config --global color.branch auto
git config --global remote.origin.push refs/heads/*:refs/for/*

echo Your Config Now :
echo 

git config -l

echo 'Now you can push your changes to master'
echo 

elif [ "$1" == "pull" ]
then 
fullname_old=`git config --get user.name`
fullname=
read -p "Your Full Name In 192.168.4.64:8087[$fullname_old?]: " fullname
if [ -z "$fullname" ]; then
    fullname="$fullname_old"
fi
git config --global user.name "$fullname"

echo

sed -i '7d' .git/config
sed '7 iurl = ssh://'$fullname'@192.168.4.64:29418/repo1' -i .git/config

email_old=`git config --get user.email`
read -p "Your Email  In 192.168.4.64:8087[$email_old?]: " email
if [ -z "$email" ]; then
    email="$email_old"
fi
git config --global user.email "$email"
echo

git config --global color.diff auto
git config --global color.status auto
git config --global color.branch auto
git config --global remote.origin.push refs/heads/*:refs/for/*

echo Your Config Now :
echo 

git config -l

echo 'Now you can pull from master'
echo 

else 
echo "please make sure your options is push or pull"
fi
```