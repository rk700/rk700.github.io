---
title: 用QEMU模拟ARM环境
author: rk700
layout: post
tags:
  - linux
  - ARM
---

之前遇到ARM逆向的入门题，但没有环境只能静态分析，非常慢；而且我觉得边调试边学应该会快一些，于是决定模拟一个ARM环境装linux系统。

首先是QEMU，由于单位电脑系统是RHEL 6，没有`qemu-system-arm`，于是我不得不编译了一次。我下的是qemu-2.0.2，运行

<pre>
$ ./configure --target-list="arm-softmmu arm-linux-user" --enable-sdl --prefix=/usr
$ make
$ make install
</pre>

来安装QEMU。

然后创建一个qcow2镜像文件，我分了10G

<pre>$ qemu-img create -f qcow2 arm.qcow2 10G</pre>

接下来就该在上面装系统了，[这里](https://gist.github.com/bdsatish/7476239)的指示写的很详细，用的是Debian 7。但启动还需要`vmlinux`和`initrd`，在[这里](http://ftp.debian.org/debian/dists/wheezy/main/installer-armhf/current/images/vexpress/netboot/)下载。运行：

<pre>$ qemu-system-arm -m 1024M -sd arm.qcow2 -M vexpress-a9 -cpu cortex-a9 -kernel ../Downloads/vmlinuz-3.2.0-4-vexpress -initrd ../Downloads/initrd.gz -append "root=/dev/ram" -no-reboot</pre>

就进入到安装界面了，稍微有点慢，一个多小时安装完成。

安装结束后，还需要把装上的`vmlinuz`和`initrd`提取出来，传给QEMU启动。提取时需要挂载我们的qcow2镜像，要加载nbd模块，结果我的RHEL 6又悲剧了……然后比较二的是在虚拟机里的ubuntu 14里完成这些工作，再拷回RHEL。命令如下

<pre>
$ modprobe nbd max_part=16
$ qemu-nbd -c /dev/nbd0 arm.qcow2
$ mount /dev/nbd0p1 /mnt
$ cp /mnt/* ~
$ umount /mnt
$ qemu-nbd -d /dev/nbd0
</pre>

因为安装时分区`boot`分在了第一个分区，所以`mount`时用`nbd0p1`。

有了提取出来的`vmlinuz-3.2.0-4-vexpress`和`initrd.img-3.2.0-4-vexpress`，终于可以启动了：

<pre>$ qemu-system-arm -m 1024M -sd arm.qcow2 -M vexpress-a9 -cpu cortex-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -append "root=/dev/mmcblk0p2"</pre>

这里用`mmcblk0p2`，是还是因为分区时`/`分在了第二个分区。

以后还是用ssh连上去，也不需要显示窗口，所以用

<pre>$ qemu-system-arm -m 1024M -sd arm.qcow2 -M vexpress-a9 -cpu cortex-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -append "root=/dev/mmcblk0p2" -display none -redir tcp:33333::22 &</pre>

让本地的`33333`端口转到ssh。需要连上去时就用

<pre>$ ssh roo@127.0.0.1 -p33333</pre>
