---
title: The Last Hope
author: rk700
layout: post
categories:
  - writeup
tags:
  - exploit
  - wechall
---
<http://www.wechall.net/challenge/bsdhell/thelasthope/index.php>  
这道题给了一个32位的ELF文件

首先试着运行，发现停住了，从IDA反汇编得到的代码可以得知有反调试之类的措施，于是我们必须先去掉那些干扰

参考这两篇文章：  
<http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html>  
<http://www.exploit-db.com/papers/13234/>  
在调用`main`之前会调用`.ctors`中的函数。于是我们先查看`.ctors`:  

{% highlight bash %}
lrk@laptop:~/tmp$ objdump -s -j .ctors ./bsd_thelasthope.elf
./bsd_thelasthope.elf:     file format elf32-i386
Contents of section .ctors:
 804af00 ffffffff 5b8c0408 00000000           ....[.......
{% endhighlight %}

其中`0x08048c5b`那里的函数会被调用，而检查发现那个函数是`anti_ptrace`，于是我们将`0x0804af04`处的值改为`0xffffffff`。这样就不会在`__do_global_ctors_aux`里调用`anti_ptrace`了

接下来，看反汇编得到的代码  
<pre>
.text:08048D59                 call    anti_ptrace
.text:08048D5E                 mov     dword ptr [esp+4], offset handler ; handler
.text:08048D66                 mov     dword ptr [esp], 5 ; sig
.text:08048D6D                 call    _signal
</pre>

同样，我们不要调用`anti_ptrace`，所以将`0x08048d59`处改为`0x90` NOP；  
另一方面，5是SIGTRAP的值，这是调试器用的，`signal`会将一个空函数注册到那个事件，我们不想那样，于是将`0x08048d6d`处也改为NOP

接下来还有反调试的代码：  
<pre>
.text:08048DA1                 mov     dword ptr [esp+0Ch], 0
.text:08048DA9                 mov     dword ptr [esp+8], 1
.text:08048DB1                 mov     dword ptr [esp+4], 0
.text:08048DB9                 mov     dword ptr [esp], 0 ; request
.text:08048DC0                 call    _ptrace
.text:08048DC5                 test    eax, eax
.text:08048DC7                 jns     short loc_8048DE1
</pre>
这里我是将`call _ptrace`改为`xor eax,eax`了

到此反调试的代码基本就解决了，然后是得到username和password

通过读几个函数，我们知道username的长度<=6，不包含大写字母，  
<pre>
U[3]%5 == 0 (candidates: 65, 70, 75, 80, 85, 90)
2^(U[2]-1)mod(U[2])==1 (prime number?)
2^(U[5]-1)mod(U[5])==1 (prime number?)
total = U[2]+U[3]+U[5]+13(U[0]+U[1]+U[4]) = 3285
U[0]=87
U[1]=72
U[4]=77
</pre>

尝试几个素数，得到一个可行解为whoami

password要简单一些:  
<pre>
password: 13>=len>8 (len==10?)
l=[31,10,30,17,11,9,25,15,1,20,22,12,6,13,101]
encrypt: p ^ l => P
chomp: remove last byte(new line)
chomp(P)==Oxw|n]nfog
</pre> 
得到password为PrimeTwins