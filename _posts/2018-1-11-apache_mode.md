---
layout: post
title: "apache prefork 下StartServers、MinSpareServers、MaxSpareServers等选项的关系"
categories: web服务器
tags:  [apache]  
author: sunzhen
---

有三种稳定的MPM（Multi-Processing Module，多进程处理模块）模式。它们分别是prefork，worker和event，它们同时也代表这Apache的演变和发展。

## prefork

首先，prefork的每个子进程只有一个线程。控制进程在建立“StartServers”个子进程后，当未满足MinSpareServers设置的进程数时，在第一个单位时间，继续创建1个子进程，再等待一个单位时间，继续创建两个……如此按指数级增加创建的进程数，最多达到每秒32个，当达到每秒32个子进程的时候就不会再指数增加了。MaxSpareServers设置了最大的空闲进程数，如果空闲进程数大于这个 值，Apache会自动kill掉一些多余进程。这个值不要设得过大，但如果设的值比MinSpareServers小，Apache会自动把其调整为 MinSpareServers+1。如果站点负载较大，可考虑同时加大MinSpareServers和MaxSpareServers。每个子进程只有一个线程，在一个时间点内，只能处理一个请求。

优点：

- 进程独立，稳定
- 没有线程安全问题

缺点：

- 进程占用系统资源过大
- 一个进程只有一个线程处理高并发不理想

下面看一下prefork具体进程分配方式，配置：

```config
<IfModule mpm_prefork_module>  
StartServers 2  
MinSpareServers 3  
MaxSpareServers 5  
MaxRequestWorkers 250  
MaxConnectionsPerChild 0  
</IfModule>  
```
重启apache后，查看进程数：
```shell
# ps aux|grep httpd |grep -v grep  
root       6878  0.2  0.4 307700 11784 ?        Ss   14:56   0:00 /usr/sbin/httpd -DFOREGROUND  
apache     6879  0.0  0.2 322048  6512 ?        S    14:56   0:00 /usr/sbin/httpd -DFOREGROUND  
apache     6880  0.0  0.2 322048  6512 ?        S    14:56   0:00 /usr/sbin/httpd -DFOREGROUND  
apache     6955  0.0  0.2 322048  6512 ?        S    14:56   0:00 /usr/sbin/httpd -DFOREGROUND 
```
只有三个子进程。

验证最大只能到单位时间开启32个子进程，配置：

```config
<IfModule mpm_prefork_module>  
StartServers 2  
MinSpareServers 90  
MaxSpareServers 120  
MaxRequestWorkers 250  
MaxConnectionsPerChild 0  
</IfModule>  
```

这个配置的意思是，起始两个apache子进程，然后2+1+2+4+8+....+32 = 65个子进程，若是下一个时间单位开启64个子进程的话，65+64=129大于120，多余的进程会被kill吊，最后稳定的进程数应该是120个，若仍然是32个的话，65+32=97最后稳定的进程数应该是97个。
重启apache后等一段时间查看进程：

```shell
# ps aux|grep httpd |grep -v grep |wc -l  
98  
```
除去root进程的话刚好97个，最高32个得证。

## worker

使用了多进程+多线程的模式。它也预先fork了几个子进程（数量比较少），每个子进程能够生成一些服务线程和一个监听线程，该监听线程监听接入请求并将其传递给服务线程处理和应答。

优点：
- 系统资源占用少
- 多线程在高并发下表现优秀

缺点：
- 在多个keep-alive下，尽管HTTP的Keepalive方式能减少TCP连接数量和网络负载，但是 Keepalive需要和服务进程或者线程绑定，这就导致一个繁忙的服务器会耗光所有的线程
- 要考虑线程安全问题

## event

event和worker工作模式答题差不多。不同的是，event模式下有一个专门的线程来管理这些 keep-alive 类型的线程，当有真实请求过来的时候，将请求传递给服务线程，执行完毕后，又允许它释放。