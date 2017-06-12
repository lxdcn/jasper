---
layout: post
cover: assets/images/cover.jpg
subclass: post
title: 受限环境中的奇淫技巧之 — ssh通道 
navigation: true
logo: assets/images/host.png
date: 2016-08-15 12:34:56
categories: lxd
tags: tech ssh tunnel port-forwarding chs
comments: true
---

### ssh 端口转发
先复习一下基础知识，ssh 端口转发（ssh Port Forwarding），也叫ssh通道（ssh tunnel），是openSSH提供的~~中国~~特色功能。它的功能，是在一条已经建立的ssh连接的基础上，将对本地端口的请求，经由ssh，通过ssh server转发到另外一台服务器；或者对ssh server端口的请求经由ssh，通过本地机器转发到另外一台服务器。它有3种：

- ssh本地端口转发（Local port forwarding）
- ssh远端端口转发（Remote port forwarding）
- ssh动态端口转发（Dynamic port forwarding）

假定有一个程序员叫Alice，她有一台笔记本（laptop），还有一个在云提供商上的Linux server（workstation），她平时ssh连接到workstation上是这样操作的：`[alice@laptop ~] $ ssh -p 22 alice@workstation` —— Alice在她的笔记本（laptop）上，用ssh默认端口22，连接到她的workstation上。

**ssh本地端口转发**，是将Alice对laptop的某个端口的TCP请求，通过ssh，经由workstation，转发到另外一台server的特定端口上，靠ssh的`-L`开关 ，命令是这样的：`[alice@laptop ~] $ ssh -L 3000:server:4000 -p 22 alice@workstation`。如是，Alice在laptop上访问3000端口，就相当于<u>在workstation上访问server的4000端口</u>。比如有个HTTP服务Alice在laptop上访问不到，但是可以在workstation上访问到，借助`ssh -L 8080:server:80 -p 22 alice@workstation`本地端口转发，Alice在laptop上用浏览器访问`http://localhost:8080`，相当于在workstation上访问`http://server`。

再比如Alice还有一台更早的Linux server，但由于常年用来翻墙，已经被功夫网封禁掉了（就叫workstation_blocked），除了先手动ssh到workstation上再ssh到workstation_blocked上之外，她还可以先执行：`[alice@laptop ~] $ ssh -L 2000:workstation_blocked:22 alice@workstation`，然后每次需要访问workstation_blocked的时候执行`[alice@laptop ~] $ ssh alice@localhost -p 2000`就可以了。——这个就是ssh中继（ssh relay）。

**ssh远端端口转发**，和本地端口转发相反。Alice建立了ssh连接之后，使得对workstation上某个端口的TCP访问，经由ssh，通过laptop，转发到另外一台server上。靠的是ssh的`-R`开关，命令是这样的：`[alice@laptop ~] $ ssh -R 3000:server:4000 -p 22 alice@workstation`。如是，Alice在workstation上访问3000端口，就相当于在laptop上访问server的4000端口。

无论是本地端口转发还是远端端口转发，server都可以是localhost，这个对远端端口转发来说就很有用。比如有这样的场景：Alice的laptop平时搁家里，连无线路由Wi-Fi上网，Alice在公司上班的时候，忙里偷闲，想通过公司的PC访问远在家里的laptop怎么办？首先，Alice在早上上班出门前需要先在laptop上用ssh连接到workstation上，打开远端端口转发到laptop的22端口：`[alice@laptop ~] $ ssh -R 2000:localhost:22 -p 22 alice@workstation`，这样，在workstation上对2000端口的TCP连接，都会经由ssh，通过laptop，转发到laptop本身（localhost）的22端口上。然后她在公司的PC上，先ssh普通连接到workstation，再`[alice@workstation ~] $ ssh -p 2000 alice@localhost`，就可以偷闲了。

**ssh动态端口转发**，即在ssh连接上开启一个SOCKS的代理端口，是翻墙3大法器（VPN、HTTP Proxy、SOCKS Proxy）中SOCKS Proxy的ssh分支。命令是`[alice@laptop ~] $ ssh -D 1080 -C -p 22 alice@server_ip`，然后在浏览器中设置SOCKS代理到127.0.0.1，1080端口，即可翻墙。

openSSH还提供了两个开关可以结合端口转发使用，`-N`在ssh连接成功后不开启shell，`-f`在ssh连接成功后会把ssh搁到后台。

---

### 实际场景
复习完基础知识，就到实际场景应用了，既然是受限环境，那各式各样政策规定和陈腐的基础设施，就会导致各种奇葩的问题需要解决。事实上，我们这些软件从业者工作在一个纯软件堆砌的工作平台上，可以说软件，或者说技术，理论上可以解决一切问题，前提是成本的考量和政策规范的允许。当软件因故做不到一些事情的时候，政策规定和游戏规则可以来弥补；当博弈成本太高不堪重负的时候，软件可以来帮忙。这儿其实在做第3类事情：技术救场，算是局部优化，虽然有效，终究不是正途。

场景是这样的，有这样1台服务器，我们称它为workstation，它分别可以ssh到另外2台服务器上去，我们分别称为A和B。workstation可以分别和A、B ssh连接，但是反之则不行，而A和B之间是网络隔离的，现在我们需要让A可以从B获取数据，比如B上面开启一个HTTP Server，让A能够访问。哦，对了， workstation还是一台Windows Server，呵呵。

首先需要在Windows上能够进行ssh操作，并且可以打开端口转发，精品小工具Putty就可以完成这个操作，只不过它的[tunnels设置](https://howto.ccs.neu.edu/howto/windows/ssh-port-tunneling-with-putty/)不如命令行明快，如果内心深处对命令行/Linux有追求，可以安装Cygwin，Git Bash等工具。

有了前面Local Port Forwarding和Remote Port Forwarding的知识准备，串联起来就可以达到效果，假定serverB上面开启了一个4000端口的HTTP Server，分别启用下面两个端口转发：

- `[alice@workstation ~] $ ssh -R 2000:localhost:3000 alice@serverA`
- `[alice@workstation ~] $ ssh -L 3000:localhost:4000 alice@serverB`

那么在serverA上访问本地端口2000，就可以访问到serverB的4000。


这个场景可以再扩展一点点，假定serverA所在的机房有个集群，除了serverA还有serverA1、serverA2、serverA3 ……，这些serverAn网络互通，并且都需要访问serverB上端口4000的HTTP Server，怎么办？
首先，针对ssh的远端端口转发，绑定在远端host上的那个端口，（比如`[alice@workstation ~] $ ssh -R 2000:localhost:3000 alice@serverA`中的2000），默认只接受来自localhost的请求。想要破坏掉这一点，修改`/etc/ssh/sshd_config`配置文件中的`GatewayPorts`为`yes`，重新加载sshd服务就可以了，这样，这些serverAn机器，统一访问serverA上的2000端口，即可以访问到serverB的4000端口。

---

### Refers
- https://help.ubuntu.com/community/SSH/OpenSSH/PortForwarding







