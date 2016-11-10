---
title: pwnable.kr之uaf
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - exploit
  - pwnable.kr
---

由这道题目名称便可知，是需要使用Use After Free(UAF)。漏洞很明显，我们只需要首先free掉空间，再分配并向释放的地址写入内容，最后再次use即可。

具体地，在use时会调用类的虚方法`introduce()`，根据反汇编可知，该方法在虚表中是第二项，而第一项则是我们要调用的虚方法`give_shell()`。所以，我们需要向对象的虚表处写入的内容，是原虚表地址的前一项（即-0x8处）。从而在调用类的第二项虚方法时，实际调用的是原虚表的第一项`give_shell()`。

接下来，问题就是再次分配多大的内容。还是通过反汇编，知道为类`Man`和`Woman`分配的chunk的大小都是0x20。而这种小的chunk是属于fastbin的。fastbin具有后进先出(LIFO)的特性，由于在free时是首先释放对象`m`再释放`w`，所以如果我们申请两次大小为0x20的chunk，那么得到的分别是之前对象`w`和`m`的地址。

一开始我没有考虑到LIFO，只重新分配了一次内存，写入的是原本`w`的地址，所以在最后use时，访问的还是没有写入的`m`，不能执行我们的目标。

为了要分配大小为0x20的chunk，我们可以new一块大小为8 bytes至24 bytes的区域。连续两次执行after，将我们伪造的虚表地址写入对象`m`和`w`的虚表处。

最后，再执行use，便会调用`give_shell()`方法，从而得到shell。

