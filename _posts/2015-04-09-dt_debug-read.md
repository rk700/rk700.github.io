---
title: 通过DT_DEBUG来获得各个库的基址
author: rk700
layout: post
redirect_from: 
  - /article/2015/04/09/dt_debug-read
  - /article/2015/04/09/dt_debug-read/
tags:
  - linux
  - exploit
  - binary
---

最近，在学习BCTF和0CTF的writeup时，注意到了一种通过`DT_DEBUG`来获得库的基址的方式：BCTF里的pattern用这一方法来获得ld-linux.so的地址，0CTF里的sandbox用这一方法来获得sandbox.so的基址。之前面对ASLR，我只知道可以通过GOT来获取libc.so的地址，而其他库的地址还不清楚应该怎样取得。于是，我稍微研究了下，在此记录。

首先，通过`readelf -d`，可以得到`.dynamic`的信息。而有些二进制文件里的`.dynamic`里包含`DT_DEBUG`：

<pre>
Dynamic section at offset 0x7c8 contains 20 entries:
  Tag        Type                         Name/Value
  ...
  0x0000000000000015 (DEBUG)              0x0
  ...
</pre>

这里`DT_DEBUG`的值是0。在实际运行时，`DT_DEBUG`的值是指向`struct r_debug`的指针。其定义如下：

{% highlight c %}
/* Rendezvous structure used by the run-time dynamic linker to communicate
   details of shared object loading to the debugger.  If the executable's
   dynamic section has a DT_DEBUG element, the run-time linker sets that
   element's value to the address where this structure can be found.  */

struct r_debug
  { 
    int r_version;              /* Version number for this protocol.  */

    struct link_map *r_map;     /* Head of the chain of loaded objects.  */

    /* This is the address of a function internal to the run-time linker,
       that will always be called when the linker begins to map in a
       library or unmap it, and again when the mapping change is complete.
       The debugger can set a breakpoint at this address if it wants to
       notice shared object mapping changes.  */
    ElfW(Addr) r_brk;
    enum
      { 
        /* This state value describes the mapping change taking place when
           the `r_brk' address is called.  */
        RT_CONSISTENT,          /* Mapping change is complete.  */
        RT_ADD,                 /* Beginning to add a new object.  */
        RT_DELETE               /* Beginning to remove an object mapping.  */
      } r_state;

    ElfW(Addr) r_ldbase;        /* Base address the linker is loaded at.  */
  };
{% endhighlight %}

可以看到，其第二个元素是指向`struct link_map`的指针。其定义如下：

{% highlight c %}
/* Structure describing a loaded shared object.  The `l_next' and `l_prev'
   members form a chain of all the shared objects loaded at startup.

   These data structures exist in space used by the run-time dynamic linker;
   modifying them may have disastrous results.  */

struct link_map
  {
    /* These first few members are part of the protocol with the debugger.
       This is the same format used in SVR4.  */

    ElfW(Addr) l_addr;          /* Difference between the address in the ELF
                                   file and the addresses in memory.  */
    char *l_name;               /* Absolute file name object was found in.  */
    ElfW(Dyn) *l_ld;            /* Dynamic section of the shared object.  */
    struct link_map *l_next, *l_prev; /* Chain of loaded objects.  */
  };
{% endhighlight %}

于是，遍历link_map，对比`l_name`，找到目标之后，就可以通过`l_addr`获得那个库的基址。

实例如下，比如说我们要找ld-linux.so的基址。首先，检查`.dynamic`的内容：

{% highlight bash %}
gdb-peda$ x/20gx 0x6007c8
0x6007c8:       0x0000000000000001      0x0000000000000010
0x6007d8:       0x000000000000000c      0x0000000000400400
0x6007e8:       0x000000000000000d      0x00000000004006d8
0x6007f8:       0x000000006ffffef5      0x0000000000400260
0x600808:       0x0000000000000005      0x0000000000400310
0x600818:       0x0000000000000006      0x0000000000400280
0x600828:       0x000000000000000a      0x000000000000004b
0x600838:       0x000000000000000b      0x0000000000000018
0x600848:       0x0000000000000015      0x00007f38cd4bd140
0x600858:       0x0000000000000003      0x0000000000600960
{% endhighlight %}

`DT_DEBUG`是0x15，所以`0x600848`那里就是`DT_DEBUG`的条目，其值是`0x00007f38cd4bd140`，即`struct r_debug`的地址。

{% highlight bash %}
gdb-peda$ x/6x 0x00007f38cd4bd140
0x7f38cd4bd140 <_r_debug>:      0x0000000000000001      0x00007f38cd4bd168
0x7f38cd4bd150 <_r_debug+16>:   0x00007f38cd2aba90      0x0000000000000000
0x7f38cd4bd160 <_r_debug+32>:   0x00007f38cd29c000      0x0000000000000000
{% endhighlight %}

对照定义，我们知道第二个元素，`0x00007f38cd4bd168`是link_map链表的第一个元素的地址。

{% highlight bash %}
gdb-peda$ x/6x 0x00007f38cd4bd168
0x7f38cd4bd168: 0x0000000000000000      0x00007f38cd2b69fb
0x7f38cd4bd178: 0x00000000006007c8      0x00007f38cd4bd700
0x7f38cd4bd188: 0x0000000000000000      0x00007f38cd4bd168
gdb-peda$ x/s 0x00007f38cd2b69fb
0x7f38cd2b69fb: ""
{% endhighlight %}

而这里第二个元素`l_name`是空，不是我们要找的库。于是通过第四个元素`l_next`，即`0x00007f38cd4bd700`，来看下一个

{% highlight bash %}
gdb-peda$ x/6x 0x00007f38cd4bd700
0x7f38cd4bd700: 0x00007fff09fe9000      0x00007f38cd2b69fb
0x7f38cd4bd710: 0x00007fff09fe9318      0x00007f38cd4ba658
0x7f38cd4bd720: 0x00007f38cd4bd168      0x00007f38cd4bd700
gdb-peda$ x/s 0x00007f38cd2b69fb
0x7f38cd2b69fb: ""
{% endhighlight %}

同理，这里也不是。继续看下一个`0x00007f38cd4ba658`

{% highlight bash %}
gdb-peda$ x/6x 0x00007f38cd4ba658
0x7f38cd4ba658: 0x00007f38ccede000      0x00007f38cd4ba640
0x7f38cd4ba668: 0x00007f38cd294b40      0x00007f38cd4bc998
0x7f38cd4ba678: 0x00007f38cd4bd700      0x00007f38cd4ba658
gdb-peda$ x/s 0x00007f38cd4ba640
0x7f38cd4ba640: "/lib64/libc.so.6"
{% endhighlight %}

这里是libc.so，那么我们继续看下一个`0x00007f38cd4bc998`

{% highlight bash %}
gdb-peda$ x/6x 0x00007f38cd4bc998
0x7f38cd4bc998 <_rtld_local+2456>:      0x00007f38cd29c000      0x0000000000400200
0x7f38cd4bc9a8 <_rtld_local+2472>:      0x00007f38cd4bbe10      0x0000000000000000
0x7f38cd4bc9b8 <_rtld_local+2488>:      0x00007f38cd4ba658      0x00007f38cd4bc998
gdb-peda$ x/s 0x0000000000400200
0x400200:       "/lib64/ld-linux-x86-64.so.2"
{% endhighlight %}

OK，这里就是ld-linux.so了。`l_addr`的值是`0x00007f38cd29c000`，我这里开了ASLR，而ld-linux.so的基址就正好是`0x00007f38cd29c000`

{% highlight bash %}
gdb-peda$ vmmap
Start              End                Perm      Name
0x00400000         0x00401000         r-xp      /root/heap.out
0x00600000         0x00601000         rw-p      /root/heap.out
0x00007f38ccede000 0x00007f38cd092000 r-xp      /usr/lib64/libc-2.18.so
0x00007f38cd092000 0x00007f38cd291000 ---p      /usr/lib64/libc-2.18.so
0x00007f38cd291000 0x00007f38cd295000 r--p      /usr/lib64/libc-2.18.so
0x00007f38cd295000 0x00007f38cd297000 rw-p      /usr/lib64/libc-2.18.so
0x00007f38cd297000 0x00007f38cd29c000 rw-p      mapped
0x00007f38cd29c000 0x00007f38cd2bc000 r-xp      /usr/lib64/ld-2.18.so
0x00007f38cd4ae000 0x00007f38cd4b1000 rw-p      mapped
0x00007f38cd4ba000 0x00007f38cd4bb000 rw-p      mapped
0x00007f38cd4bb000 0x00007f38cd4bc000 r--p      /usr/lib64/ld-2.18.so
0x00007f38cd4bc000 0x00007f38cd4bd000 rw-p      /usr/lib64/ld-2.18.so
0x00007f38cd4bd000 0x00007f38cd4be000 rw-p      mapped
0x00007fff09f9f000 0x00007fff09fc0000 rw-p      [stack]
0x00007fff09fe7000 0x00007fff09fe9000 r--p      [vvar]
0x00007fff09fe9000 0x00007fff09feb000 r-xp      [vdso]
0xffffffffff600000 0xffffffffff601000 r-xp      [vsyscall]
{% endhighlight %}

由此，我们得到了ld-linux.so的地址。同样的道理，我们通过遍历link_map，就可以得到所有库的地址。当然，前提是二进制文件需要有`DT_DEBUG`。

**Edit:** 最近研究才意识到，不一定需要从`DT_DEBUG`去获得`link_map`的地址。事实上，`.got.plt`的前3项，分别是`.dynamic`的地址，`link_map`的地址和`_dl_runtime_resolve`的地址。在解析函数地址时，`link_map`会被作为参数推到栈上传递给`_dl_runtime_resolve`。关于这一过程及利用方式，详情可见[http://rk700.github.io/article/2015/08/09/return-to-dl-resolve](http://rk700.github.io/article/2015/08/09/return-to-dl-resolve)
