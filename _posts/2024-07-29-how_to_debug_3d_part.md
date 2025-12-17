---
layout: post
title: Linux 第三方库遇到 bug 怎么办
date: 2024-07-29
tags:
  - linux
comments: true
author: sgarifool
toc: false
---

> 本文记录了如何在写 Linux 代码时 debug 第三方库

<!-- more -->

最近在 Linux 上写代码, 发现刚刚写好的程序在运行的时候发生了 `segmentation fault` (段错误), 在使用 vscode 的 Debug 功能进行调试代码的时候, 发现程序停在了一句使用 librtmp 库的 API 的代码上

```cpp
int nRet = RTMP_SendPacket(m_pRtmp,packet,TRUE);
```

vscode 只告诉我这句话发生了段错误, 按理来说应该是我传入的两个指针 `m_pRtmp` 和 `packet` 可能出了问题, 但是这代码逻辑我使用的是雷霄骅大佬的源码改的, 而且逻辑单独拿出来跑也是没有任何问题的, 这就导致我非常想知道在这句代码内部到底发生了什么事情?!

---

不过由于使用的是第三方库, 我现在只能看到这个库的头文件, 而源码早就已经编译到了库文件里, vscode 上调试的进入函数根本无法进入到这个库的源码里. 不过使用 vscode 调试的本身就是一个带有调试信息的可执行文件, 那我在连接这个库的时候直接链接一个带有调试信息的库就应该就可以成功进入调试了

---

理论存在,实践开始。我去 [rtmp 库官网](https://rtmpdump.mplayerhq.hu/)找到了 librtmp 库的源代码, 目录结构如下所示: 

```shell
rtmpdump/
├── ChangeLog
├── COPYING
├── librtmp
│   ├── amf.c
│   ├── amf.h
│   ├── bytes.h
│   ├── COPYING
│   ├── dhgroups.h
│   ├── dh.h
│   ├── handshake.h
│   ├── hashswf.c
│   ├── http.h
│   ├── librtmp.3
│   ├── librtmp.3.html
│   ├── librtmp.pc.in
│   ├── log.c
│   ├── log.h
│   ├── Makefile
│   ├── parseurl.c
│   ├── rtmp.c
│   ├── rtmp.h
│   └── rtmp_sys.h
├── Makefile
├── README
├── rtmpdump.1
├── rtmpdump.1.html
├── rtmpdump.c
├── rtmpgw.8
├── rtmpgw.8.html
├── rtmpgw.c
├── rtmpsrv.c
├── rtmpsuck.c
├── thread.c
└── thread.h
```

可以发现这是一个使用 Makefile 来编译构建的第三方库, 一般的安装流程是: 

```shell
cd rtmpdump
sudo make
sudo make install
```

而这样直接编译出来的库是没有包含调试信息的

所以需要在 make 之前, 更改 Makefile 文件, 使得编译出来的库包含调试信息

打开 Makefile 文件 (源码里有两个 Makefile 文件,我都更改了) 在文件里寻找 `CFLAGS` 字段和 `OPT` 字段并对这两个地方进行修改

```shell
# OPT=-O2
OPT=-O0
# CFLAGS=-Wall $(XCFLAGS) $(INC) $(DEF) $(OPT) $(SO_DEF)
CFLAGS=-g -O0 -Wall $(XCFLAGS) $(INC) $(DEF) $(OPT) $(SO_DEF)
```

将原来的 `-O2` 优化变成 `-O0` , 标识在编译的时候不优化代码, 并且在 `CFLAGS` 后面加入了 `-g` 表示生成可调式的文件, 在更改了 Makefile 文件之后执行 make 命令就可以生成带调试信息的 rtmp 库的文件了, make 后文件目录变成如下: 

```shell
rtmpdump/
├── ChangeLog
├── COPYING
├── librtmp
│   ├── amf.c
│   ├── amf.h
│   ├── amf.o
│   ├── bytes.h
│   ├── COPYING
│   ├── dhgroups.h
│   ├── dh.h
│   ├── handshake.h
│   ├── hashswf.c
│   ├── hashswf.o
│   ├── http.h
│   ├── librtmp.3
│   ├── librtmp.3.html
│   ├── librtmp.a
│   ├── librtmp.pc.in
│   ├── librtmp.so -> librtmp.so.1
│   ├── librtmp.so.1
│   ├── log.c
│   ├── log.h
│   ├── log.o
│   ├── Makefile
│   ├── parseurl.c
│   ├── parseurl.o
│   ├── rtmp.c
│   ├── rtmp.h
│   ├── rtmp.o
│   └── rtmp_sys.h
├── Makefile
├── README
├── rtmpdump
├── rtmpdump.1
├── rtmpdump.1.html
├── rtmpdump.c
├── rtmpdump.o
├── rtmpgw
├── rtmpgw.8
├── rtmpgw.8.html
├── rtmpgw.c
├── rtmpgw.o
├── rtmpsrv
├── rtmpsrv.c
├── rtmpsrv.o
├── rtmpsuck
├── rtmpsuck.c
├── rtmpsuck.o
├── thread.c
├── thread.h
└── thread.o
```

发现在 `librtmp` 文件夹下生成了 `librtmp.a` 等可执行文件

于是我将 `librtmp.a` 这个文件链接到了我自己编写的代码中, 虽然如果仅仅只链接这个文件好像还是会报错, 因为这个这个库在实现的时候还使用了其它的库如 openssl 和 zlib 库,所以还需要把这些库也链接上, 秉着有报错问 chat 的原则, 终于将这个带有调试信息的 rtmp 库链接到了我自己写的程序中

使用vscode调试也成功进入了函数的内部, 看到了究竟是哪一句话发生了非法读写
