---
title: Linux的capabilities机制
author: rk700
layout: post
catalog: true
tags:
  - linux
---

在传统的Linux的权限控制机制中，SUID(Set User ID on execution)是一个比较有意思的概念。例如，文件`/bin/passwd`设置了SUID的flag，所以普通用户在运行`passwd`命令时，便以`passwd`的所属者，即root用户的身份运行，从而修改密码。

但是，这也带来了安全隐患。我们运行SUID的命令时，通常只是需要使用一小部分特权，但是使用SUID，却可以拥有root用户的全部权限。所以，一旦SUID的文件存在漏洞，便可能被利用，以root身份执行其他操作。因此，SUID机制，在某种程度上增大了安全攻击面。

为了缓解SUID带来的安全风险，设置SUID需要谨慎，而且这些二进制文件也会通过降低自身euid，并只在需要特权时再恢复为root，以此来缩小以root身份运行的窗口。

而仔细反思，我们便可以看到，SUID的问题，主要在于权限控制太粗糙。为了对root身份进行更加精细的控制，Linux增加了另一种机制，即capabilities。本文，便将对这一机制进行介绍。

---

## Capabilities简介

Capabilities机制，是在Linux内核2.2之后引入的。它将root用户的权限细分为不同的领域，可以分别启用或禁用。从而，在实际进行特权操作时，如果euid不是root，便会检查是否具有该特权操作所对应的capabilities，并以此为依据，决定是否可以执行特权操作。

例如，下表列出了一些常见的特权操作及其对应的capability：

|改变文件的所属者(chown()) | CAP_CHOWN|
|向进程发送信号(kill(), signal()) | CAP_KILL|
|改变进程的uid(setuid(), setreuid(), setresuid()等) | CAP_SETUID|
|trace进程(ptrace()) | CAP_SYS_PTRACE|
|设置系统时间(settimeofday(), stime()等) | CAP_SYS_TIME|

Capabilities是细分到线程的，即每个线程可以有自己的capabilities。而完整的capabilities实现，除了对线程的capabilities具有以下相关功能：

- 进行特权操作时，检查该线程是否拥有该操作的capability
- 提供系统调用，用于获取或修改线程的capability

还应该包含对于文件的capabilities的支持，即：

- 文件系统支持文件附加属性，使得可执行文件具有一定的capabilities，从而在运行时确定其capabilities

对于文件capabilities的支持，直到内核2.6.24之后才完成。

以下，将分别介绍线程（进程）的capabilities和文件的capabilities。

---

## 线程与文件的capabilities

#### 线程的capabilities

每一个线程，具有3个capabilities的集合，分别是：

- Permitted
- Inheritable
- Effective

每个集合中，可以包含零个或多个capabilities。这3种集合的具体含义如下：

**Permitted**

这个集合定义了线程所*能够*拥有的特权的上限。换句话说，如果某个capability不在Permitted集合中，那么该线程便不能进行这个capability所对应的特权操作。Permitted集合是Inheritable和Effective集合的的超集。

**Inheritable**

当执行`exec()`系运行其他命令时，能够被新命令继承的capabilities，被包含在Inheritable集合中。

**Effective**

内核检查该线程是否可以进行特权操作时，检查的对象便是Effective集合。如之前所说，Permitted集合定义了上限。线程可以删除Effective集合中的某capability，随后在需要时，再从Permitted集合中恢复该capability，以此达到临时禁用capability的功能。

(Linux 4.3之后，增加了一种集合Ambient。详情可见相关manual)

---

#### 文件的capabilities

文件的capabilities，是保存在文件的扩展属性中。修改这些扩展属性，需要具有`CAP_SETFCAP`的capability。文件与线程的capabilities，共同决定了通过`exec`运行该文件后的capabilities。

文件的capabilities功能，需要文件系统的支持。如果文件系统使用了`nosuid`选项进行挂载，那么文件的capabilities将被忽略。

类似于线程的capabilities，文件的capabilities也包含了3个集合：

**Permitted**

这个集合中包含的capabilities，在文件被执行时，被加入其Permitted集合。

**Inheritable**

这个集合与线程的Inheritable集合的交集，是执行完`exec`后实际继承的capabilities。

**Effective**

这仅仅是一个bit。如果设置开启，那么在运行`exec`后，Permitted集合中新增的capabilities会自动出现在Effective集合中；否则不会出现在Effective集合中。对于一些旧的可执行文件，由于其不会调用capabilities相关函数设置自身的Effective集合，所以可以将该可执行文件的Effective bit开启，从而将Permitted集合中的capabilities自动添加到Effective集合中。

---

#### 运行`exec`后capabilities的变化

上面介绍了线程和文件的capabilities，可能会觉得有些抽象难懂。下面将使用具体的计算公式，来说明执行`exec`后capabilities是如何确定的。

我们使用P代表执行`exec`前的capabilities，P'代表执行`exec`后的capabilities，F代表`exec`执行的文件的capabilities。那么：

P'(Permitted) = (P(Inheritable) & F(Inheritable)) \| (F(Permitted) & cap_bset)

P'(Effective) = F(Effective) ? P'(Permitted) : 0

P'(Inheritable) = P(Inheritable)

其中的cap_bset是capability bounding set。通过与文件的Permitted集合计算交集，可进一步限制某些capabilities的获取，从而降低了风险。

而正如介绍文件的Effective bit时所说，文件可以将其Effective bit关闭。由此，在通过`exec`执行该文件后，实际的Effective集合为空集。随后，在需要进行特权操作时，可再将Permitted集合中的capabilities加入Effective集合中。

## 获取和设置capabilities

系统调用`capget(2)`和`capset(2)`，可被用于获取和设置线程自身的capabilities。此外，也可以使用libcap中提供的接口`cap_get_proc(3)`和`cap_set_proc(3)`。当然，Permitted集合默认是不能增加新的capabilities的，除非CAP_SETPCAP在Effective集合中。

如果要查看线程的capabilities，可以通过`/proc/<PID>/task/<TID>/status`文件，三种集合分别对应于CapPrm, CapInh和CapEff。但这种的显示结果是数值，不适合人类阅读。为此，可使用包`libcap`中的命令`getpcaps <PID>`获取该进程的主线程的capabilities。

类似的，如果要查看和设置文件的capabilities，可以使用命令`getcap`或者`setcap`。

## 示例

#### ping

使用`getcap`命令，我们可以看到`ping`文件的capabilities:

{% highlight bash %}
root@debian:~# getcap /bin/ping
/bin/ping = cap_net_raw+ep
{% endhighlight %}

即该文件的capabilities，设置了Effective bit，而且Permitted集合中包含了CAP_NEW_RAW，从而可以发送raw packet。

#### run-as

另一个例子，是Android系统中的`run-as`命令。根据源码中的注释，`run-as`具有CAP_SETUID和CAP_SETGID，从而可以设置自身的resuid和resgid。由此，可以达到`run-as`的效果，即以特定uid运行命令。

DirtyCow漏洞在Android上的利用，便是修改`run-as`文件的内容，并利用CAP_SETUID设置resuid为root，实现权限提升。

---

## Android上的capabilities

根据[官方说明](https://source.android.com/security/enhancements/enhancements43.html)，Android 4.3之后，便不再使用SUID/SGID，而是使用capabilities机制，以减少攻击面。

但是，虽然Android源码中有包含库libcap-ng的代码，但并未编译相关代码。我们下载[libcap-ng](http://people.redhat.com/sgrubb/libcap-ng/libcap-ng-0.7.8.tar.gz)，并修改如下：

{% highlight bash %}
$ diff -u libcap-ng-0.7.8/src/cap-ng.c ~/src/libcap-ng-0.7.8/src/cap-ng.c
--- libcap-ng-0.7.8/src/cap-ng.c        2016-10-27 10:09:12.000000000 +0800
+++ /Users/liuruikai756/src/libcap-ng-0.7.8/src/cap-ng.c        2016-07-24 23:47:34.000000000 +0800
@@ -25,9 +25,7 @@
 #include <string.h>
 #include <stdarg.h>
 #include <stdio.h>
-#if !defined(ANDROID)
 #include <stdio_ext.h>
-#endif
 #include <stdlib.h>
 #include <sys/prctl.h>
 #include <pwd.h>
@@ -278,9 +276,7 @@
        f = fopen(buf, "re");
        if (f == NULL)
                return -1;
-#if !defined(ANDROID)
        __fsetlocking(f, FSETLOCKING_BYCALLER);
-#endif
        while (fgets(buf, sizeof(buf), f)) {
                if (strncmp(buf, "CapB", 4))
                        continue;
{% endhighlight %}

随后，复制Android源码中的`external/libcap-ng/config.h`和`external/libcap-ng/Android.mk`。在`config.h`中需要定义`HAVE_SYS_XATTR_H`，并修改`Android.mk`以编译`filecap`。

随后，运行命令`ndk-build`进行编译，并将相关文件push到设备：

{% highlight bash %}
$ ndk-build NDK_PROJECT_PATH=. APP_BUILD_SCRIPT=./Android.mk APP_PLATFORM=android-19
$ adb -e push libs/x86/libcap-ng.so /system/lib/
$ adb -e push libs/x86/filecap /data/local/tmp/
{% endhighlight %}

运行`/data/local/tmp/filecap`便可查看文件的capabilities了。

例如，查看之前提到的文件`/system/bin/run-as`的capabilities如下：

{% highlight bash %}
root@generic_x86:/ # /data/local/tmp/filecap /system/bin
p/filecap /system/bin/run-as                                                  <
file                 capabilities
/system/bin/run-as     setgid, setuid
{% endhighlight %}

---

**参考文献**

- [http://man7.org/linux/man-pages/man7/capabilities.7.html](http://man7.org/linux/man-pages/man7/capabilities.7.html)
- The Linux Programming Interface
- [https://www.kernel.org/pub/linux/libs/security/linux-privs/kernel-2.2/capfaq-0.2.txt](https://www.kernel.org/pub/linux/libs/security/linux-privs/kernel-2.2/capfaq-0.2.txt)
