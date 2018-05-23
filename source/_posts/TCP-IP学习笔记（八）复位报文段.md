---
title : TCP/IP学习笔记（八）复位报文段
date : 2018-03-01
categories : TCP/IP
tags : TCP/IP
---



TCP报文首部中存在一个RST位，如果该位被置1则表示这是个复位报文段。当一个报文段从一端发往一个不存在或者处于异常状态的另一端时，就会以一个复位报文段应答发送端，告知发送端连接出现错误，应当被关闭

有三种连接情况可能会产生复位报文段

* 尝试连接到一个不存在的&lt;ip,port&gt;
* 主动关闭的一方的套接字设置了SO_LINGER选项，并且超时时间为0
* 另一端异常崩溃导致连接处于半关闭状态，此时正常的一端向已关闭的一端发送报文段


<!--more-->

# 向不存在的端口发送连接请求

如果客户端尝试连接到port端口上而这个端口根本就没有服务器监听，那么当客户端发送三次握手的第一个SYN报文段时另一端会以复位报文段回应

可以在终端使用telnet ip port命令向<ip,port>发送连接请求，通过wireshark观察报文段发送情况（需要保证没有服务器监听port端口）

```shell
➜  ~ telnet localhost 9999
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
➜  ~ 
```

![](https://s1.ax1x.com/2018/03/01/9rp2t0.png)

# 主动关闭的一方设置了SO_LINGER选项

SO_LINGER选项用于对连接关闭提供更多的控制，它影响的是close函数的行为。需要配合struct linger结构使用

```c
struct linger
{
    int l_onoff;	// 开关选项
    int l_linger;	// 超时时间
};
```

使用方法为

```c
struct linger so_linger;
so_linger.l_onoff = m;
so_linger.l_linger = n;
::setsockopt(sockfd, SOL_SOCKET, SO_LINGER, &so_linger, sizeof(so_linger));
```

根据上述设置的l_oneoff和l_linger不同，SO_LINGER有三种不同的行为

* l_onoff = 0，表示关闭SO_LINGER选项，调用close函数时采用默认行为，即四次挥手，close立即返回
* l_onoff != 0，l_linger = 0，表示开启SO_LINGER选项，调用close函数时不进行四次挥手而直接发送复位报文段给对端，close立即返回
* l_onoff != 0，l_linger = 0，表示开启SO_LINGER选项，close函数变为阻塞函数，调用时会先将发送缓冲区中的数据全部发送出去并收到对端的ACK应答报文段（或者到达超时时间）再返回close函数。并且这种情况下不管套接字是否设置非阻塞close函数都会阻塞

通过wireshark观察报文段发送情况，客户端服务器的行为如下

* 建立连接，客户端设置SO_LINGER选项并设置l_onoff != 0，l_linger = 0（上述第二种情况）
* 0.5秒后客户端调用close函数终止连接
* 服务器收到复位报文段后(read/recv返回-1)输出错误信息，并调用close函数终止连接

服务器输出错误信息

```shell
➜  server ./server
Connection reset by peer
```

![](https://s1.ax1x.com/2018/03/01/9rCIoR.png)复位报文段不需要对端应答，因为RST代表连接出现错误，双方只要各自关闭就可以了

# 半关闭连接情况下发送数据

在介绍保活定时器(KEEPALIVE)时提到过，如果通信双方的一端突然崩溃，会导致连接处于半关闭状态，也就是一端已经关闭，而另一端仍处于打开状态。在这种情况下，如果正常的一端向已关闭的一端发送数据，由于对端已经重启，丢失了所有连接信息，会返回一个复位报文段

借用&lt;TCP/IP详解&gt;中的例子，在连接正常的情况下断开服务器以太网电缆并重启，随后客户端发送数据

![](https://s1.ax1x.com/2018/03/01/9r1iUx.png)

观察tcpdump结果

![](https://s1.ax1x.com/2018/03/01/9r1APK.png)

最终，服务器由于无法识别发来的连接，发送给客户端复位报文段以告知对方关闭连接




