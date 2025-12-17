---
layout: post
title: 如何在没有root权限的情况下配置软件的安装环境
date: 2025-04-21
tags:
  - linux
  - 3dpart
comments: true
author: sgarifool
toc: false
---

> 本文提供了在没有 root 权限的情况下需要安装某个软件, 但是在安装过程中报缺少某某软件库 (如 `libxcb` ) 的错误时的解决思路

<!-- more -->

## 写在前面话

上个星期帮室友在学校的云服务器上安装他们仿真所需要的 Ansys 软件, 因为他们现在需要仿真的东西是一个物理体积特别大, 结构特别多的一个模型, 据说他们实验室自己配的 I9 14900K 加上 128G 内存, 配上 RTX4090 显卡的台式机也跑不了那个模型的仿真, 于是他们找学校借了一个超算平台, 一个 96核 1000+GB 内存的平台. 但是由于学校服务器使用是 Rocky Linux 而不是 Windows, 所以他们完全不知道怎么安装, 而且学校由于安全考虑只给他们分配了一个用户, 并没有提供root权限, 他说他们去找 tb 的商家帮忙安装, 结果商家一听没有 root 秒回装不了, 找了半天好不容易找了一个说可以非 root 安装的商家, 结果人家开口要 300rmb, 他觉得不划算就跑过来找我, 问我能不能安装, 虽然最后的结果我卡在破解这一步了, 但是好歹解决了一开始连安装时环境都报错的问题, 所以还是写一个 blog 记录一下解决的过程

其实整个解决的思路非常简单, 就是找到你缺的那个环境, 然后下载到你自己账户有权限的目录, 然后将这个目录的环境添加到你当前的 shell 环境里就可以了

# 遇到的第一个报错

和大多数网上的教程一样, 我遇到的第一个报错是: 

```bash
[xxxxx@xxxxx ANSYS.2023.R1.Product.Linux64]$ ./INSTALL 
copying necessary files to /tmp/ans_install_tmp318101/
 Executing /tmp/ans_install_tmp318101/instcore
qt.qpa.plugin: Could not load the Qt platform plugin "xcb" in "" even though it was found.
This application failed to start because no Qt platform plugin could be initialized. Reinstalling the application may fix this problem.

Available platform plugins are: xcb.

/tmp/ans_install_tmp318101/INSTALL: line 611: 318161 Aborted                 (core dumped) "${fullexepath}" "$@" -style windows -launchdir "${mountDirectory}" -usetempdir "${ansTmpDir}"
```

这里告诉我们此时系统上缺少 `libxcb` 这个库, 因为 qt 找不到, 网络上的大多数方法是下一个 qt 库, 但是经过测试后发现, 其实这里的意思是我们的系统其实是有 qt 库的, 但是 qt 的插件初始化在找系统上的 `xcb` 的时候发现没有找到, 此时你安装这个库就可以了, 参考[这篇](https://www.bilibili.com/opus/857947213785989121) blog, 作者说要安装 qt, 但是其实最终还是要回到安装 `libxcb-xinerama0` 这个库上, 但是由于此时使用的系统, 没有 root 权限 (甚至外网连接的能力都没有)

我一开始的思路是去下载一些编译好了的 `libxcb` 库, 最开始找到了 [https://pkgs.org/](https://pkgs.org/) 这个网站, 通过在安装的时候指定 root 路径, 来将库安装到自己的目录下, 此时会发现

```bash
[xxxxx@xxxxx test]$ rpm -ivh --root  ~/Desktop/3dpart/ libxcb-1.13.1-1.el8.x86_64.rpm 
warning: libxcb-1.13.1-1.el8.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 6d745a60: NOKEY
error: Failed dependencies:
        libXau.so.6()(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6()(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6(GLIBC_2.14)(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6(GLIBC_2.2.5)(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6(GLIBC_2.3.2)(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6(GLIBC_2.3.4)(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        libc.so.6(GLIBC_2.4)(64bit) is needed by libxcb-1.13.1-1.el8.x86_64
        rtld(GNU_HASH) is needed by libxcb-1.13.1-1.el8.x86_64
```

发现我们所需要安装的库, 依赖了如 `libXau.so.6()(64bit)` 这样的其他库的资源, 所以会导致安装不成功

此时一种无法掌控的感觉涌上我的心头: 我还需要再去找对应的 `libXau` 库, 但是万一这两个库使用的编译环境不一样, 链接出问题了咋办. 于是我决定干脆使用源码编译的方式

# 使用源码来安装所需的环境

我在另一个网页 [https://www.linuxfromscratch.org/blfs/view/cvs/x/libxcb.html](https://www.linuxfromscratch.org/blfs/view/cvs/x/libxcb.html) 找到了 `libxcb` 的源码, 而且该页面还贴心的告诉你, 要想编译这个库, 还需要 `libXau-1.0.9` 和 `xcb-proto-1.14.1` 的依赖, 好处在于你还可以通过这个网站, 下载 `libXau-1.0.9` 和 `xcb-proto-1.14.1` 的源码 (当然, 在编译这两个库的时候还需要其他依赖, 相当于在套娃), 最后你可以通过这个链条, 找到所有需要编译的源码

在编译之前, 需要将我们自定义的第三方库的环境添加到当前的环境变量中, 假设我们此时选择将所有缺失的库都安装在 `~/Desktop/3dpart/` 这个路径下 (这里意味着会在~/Desktop/3dpart/这个目录下生成 `include, lib, share`  这三个文件夹, 注意真正在添加的时候可能需要将 ~ 换成绝对路径)

## 添加自定义的环境变量

通过编写 `.bashrc` 等配置文件, 可以实现自定义环境变量的添加

```bash
# 添加你的环境, 以 .bashrc 为例
# 打开 .bashrc
cd
vim .bashrc

# 在 .bashrc 底部添加环境
# pkgconfig 环境 
export PKG_CONFIG_PATH=~/Desktop/3dpart/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=~/Desktop/3dpart/share/pkgconfig:$PKG_CONFIG_PATH
# include 环境
export CPATH=~/Desktop/3dpart/include:$CPATH
export C_INCLUDE_PATH=~/Desktop/3dpart/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=~/Desktop/3dpart/include:$CPLUS_INCLUDE_PATH
# lib 环境
export PKG_CONFIG_PATH=~/Desktop/3dpart/lib/pkgconfig:$PKG_CONFIG_PATH
```

将上述你自定义的第三方库的环境添加到当前 shell 之后, 可以使用

```bash
source .bashrc
```

来更新你的环境变量

## 开始编译

然后我们就可以开始编译我们下载好的第三方库的源码了

对于源码编译, 所有的库都可以使用 configure 这个工具来设置安装的路径, 我们可以使用以下方式来编译安装

```bash
cd /your/soure/code/folder/
./configure --prefix=~/Desktop/3dpart/
make -j
make install
```

如果没有问题, 你已经可以在 `~/Desktop/3dpart/` 这个路径下看到 `include, lib, share`  这三个文件夹了

这里需要注意一下编译顺序, 比如 `libxcb -依赖于-> libXau && xcb-proto` 则需要先编译 `libXau && xcb-proto` 再编译 `libxcb`


