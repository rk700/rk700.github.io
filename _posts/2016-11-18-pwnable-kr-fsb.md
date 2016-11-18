---
title: pwnable.kr之fsb
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - exploit
  - pwnable.kr
---

这道题目与格式化字符串(format string)攻击相关。程序会将`argv`和`envp`清空，但是调试发现，在栈的底部仍然存在可执行文件的路径。所以基本思路就是利用符号链接。将key内存地址写入栈上。随后利用格式化字符串，获取栈上该内存地址相对于栈顶的偏移。最后通过`%x$hhx`的方式，将key的内容改写为我们设定的值。

但是，实际操作时，还是掉了几个坑。

第一个，读入字符串使用的是`read`，不会在结尾处添加`\0`。所以，如果后一次读入的字符串长度比前一次读入的少，那么在`printf`时，前一次字符串的末尾也会被打印。所以，使用完格式化字符串修改了key后，之后的读入的内容应该比之前长，防止再次利用格式化字符串写入。

第二个，在检查key时，使用的是`strtoull()`将字符串转化为`unsigned long long`。这一段对应的汇编代码是：

{% highlight nasm %}
call    _strtoull
mov     edx, eax
sar     edx, 1Fh
mov     [ebp+var_30], eax
mov     [ebp+var_2C], edx
{% endhighlight %}

而`sar edx 1Fh`这一步，说明实际的效果还是会截断的，所以传入一个很大的`unsigned long long`，最后的结果也许并不是想要的值。根据[这里](http://stackoverflow.com/questions/1646031/strtoull-and-long-long-arithmetic)的解释，如果没有包含头文件`<stdlib.h>`，那么编译器可能会将`strtoull`编译成返回`int`再扩展到`long long`。而再检查源代码，发现确实没有包含`<stdlib.h>`。所以，只看c代码，根本不知道编译器在里面又埋了什么坑。。必须要结合着汇编来分析。


