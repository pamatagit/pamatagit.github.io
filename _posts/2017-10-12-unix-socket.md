---
layout: post
title: unix socket
date: 2017-10-12
category: "linux"
---

不同主机之间的进程通信可以使用套接字（socket），这种情况是基于TCP/IP协议，需要用到IP地址。

但是同一主机上的不同进程之间通信用IP地址就有点多余，这时候可以用unix socket。

与管道相比，unix socket 既可以使用字节流，又可以使用数据队列，而管道则只能使用字节流。unix socket的接口和Internet sockert很像，但它不适用网络底层协议来通信。

unix socket使用系统文件的地址来作为自己的身份。它可以被系统进程引用。所以两个进程可以同时打开一个unix socket来进行通信，不过这种通信方式是发生在系统内核里而不会在网络里传播。