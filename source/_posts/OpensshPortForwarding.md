---
title: 基于OpenSSH的端口转发 
date: 2017-05-12 10:03:35
tags: OpenSSH
---

# OpenSSH

首先我们看一下官方对[OpenSSH的说明][1]：

> OpenSSH is the premier connectivity tool for remote login with the SSH(Secure SHell) protocol. It encrypts all traffic to eliminate eavesdropping, connection hijacking, and other attacks. In addition, OpenSSH provides a large suite of secure tunneling capabilities, several authentication methods, and sophisticated configuration options.

简单的说OpenSSH是一个连接工具，是SSH协议的一个实现。那么这个SSH优势在哪里呢？通常我们登陆到远端系统的时候有多种方式，比如常见的Telnet，但是这种登陆方式的密码和数据是明文传递的。SSH将我们所有的传输做了加密处理，这样别人即使抓包也看不到信息了。

这里提一下OpenSSL——前几年爆出漏洞，老罗还捐了不少门票的开源软件。OpenSSL提供了SSL(Secure Scket Layer)协议的开源实现。SSL是传输层和应用层的一个安全协议提供了一些加密算法，我们经常用的https就是通过SSL加密后http报文。

有一些SSH的实现是基于SSL的，毕竟加密算法SSL已经帮你实现了，用起来很方便。所以SSH可以看作是：SSH=telnet+SSL。

# OpenSSH端口转发

OpenSSH提供了诸多的工具，其中本篇文章关注的是其中两个`ssh -L`和`ssh -R`。

## 本地端口转发

本地端口转发可以将你本地的某个端口连接到服务器的端口。比如你想通过本地的8080端口连接到Ubuntu论坛，我们可以这么配置：

```shell
ssh -L 8080:www.ubuntuforums.org:80 localhost
ssh -L <local port>:<remote host>:<remote port> <SSH serverhost>
```

这是个最基本的例子，我们来看一下这个转发过程：

1. 本地访问8080端口，浏览器输入localhost:8080。
2. ssh服务一直在监听该端口，于是该请求通过ssh client端发送到ssh server端，即`<SSH serverhost>`。在这个例子里面ssh server端就是本地。
3. server接收到数据后，转发到`<remote host>:<remote port>`，在本例中就是`www.ubuntuforums.org:80`
4. 最后接收到的数据按照原路返回。

这个过程会有一点绕，主要思想就是本地向SSH Client发送数据，指定目的地址，接下来的事情就交给SSH做。

再看一个问题，有三台机器地址分别为A、B、C，现在A、C因为防火墙等原因无法直接访问，而B可以同时访问两个机器，那么A可以通过B访问C。在A中使用SSH服务：

```shell
ssh -L 1234:C:21 B 
```

C的21端口为ftp端口，因此ftp连接A的1234端口即连接C的21端口。在A中输入ftp命令：

```shell
ftp localhost:1234
```

## 远程转发

我们的例子还是上一节A、B、C三台机器，我们这一次在中转机器B中设置远程转发，远程转发的命令和本地转发相似：

```shell
ssh -R 1234:C:21 A
ssh -R <local port>:<remote host>:<remote port> <SSH serverhost>
```

因为该命令是在B机器上执行，B是SSH服务的的Client端，A是Server端。但是从应用的角度来讲，A是需要连接B的，因此A是Client端，B是Server端。在应用的Server端设置SSH连接，这种方式就是远程转发。在本例中远程转发和本地转发的效果是一样的，可以通过A的1234端口登录C机器。

## 解除转发

linux端通过kill命令解除转发：

```shell
ps aux | grep ssh
kill <id>
```

[1]: http://www.openssh.com/	"OpenSSH官方"
[2]: https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/	"实战 SSH 端口转发"
[3]: http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html	"SSH原理与运用（二）：远程操作与端口转发"

