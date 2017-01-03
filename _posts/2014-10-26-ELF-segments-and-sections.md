---
title: ELF segments and sections
author: rk700
layout: post
redirect_from:
  - /article/2014/10/26/ELF-segments-and-sections
  - /article/2014/10/26/ELF-segments-and-sections/
tags:
  - linux
---

有时候需要搞明白ELF文件里哪些部分是可写的，之前一直是稀里糊涂的，没有仔细去研究。今天稍微看了一下manpage，并动手试了下，大概明白了segments和sections之间的关系。

以下内容是基于我自己的理解写的，所以有些部分可能并不准确。

首先是两个概念，segments和sections，这两个是完全不同的。ELF文件除了ELF header，还有program header和section header：program header对应的是segments,而section header对应的是sections。

具体地，这两者之间有如下关系：

1. 每个segments可以包含多个sections
2. 每个sections可以属于多个segments
3. segments之间可以有重合的部分

首先来看sections:
<pre>
lrk@laptop:~/tmp/CTF/ISG/pwnme$ readelf -WS pwnme 
There are 28 section headers, starting at offset 0x1170:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000400238 000238 00001c 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            0000000000400254 000254 000020 00   A  0   0  4
  [ 3] .note.gnu.build-id NOTE            0000000000400274 000274 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        0000000000400298 000298 00001c 00   A  5   0  8
  [ 5] .dynsym           DYNSYM          00000000004002b8 0002b8 000090 18   A  6   1  8
  [ 6] .dynstr           STRTAB          0000000000400348 000348 000049 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          0000000000400392 000392 00000c 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         00000000004003a0 0003a0 000020 00   A  6   1  8
  [ 9] .rela.dyn         RELA            00000000004003c0 0003c0 000018 18   A  5   0  8
  [10] .rela.plt         RELA            00000000004003d8 0003d8 000078 18   A  5  12  8
  [11] .init             PROGBITS        0000000000400450 000450 00001a 00  AX  0   0  4
  [12] .plt              PROGBITS        0000000000400470 000470 000060 10  AX  0   0 16
  [13] .text             PROGBITS        00000000004004d0 0004d0 0001a2 00  AX  0   0 16
  [14] .fini             PROGBITS        0000000000400674 000674 000009 00  AX  0   0  4
  [15] .rodata           PROGBITS        0000000000400680 000680 000018 00   A  0   0  4
  [16] .eh_frame_hdr     PROGBITS        0000000000400698 000698 000034 00   A  0   0  4
  [17] .eh_frame         PROGBITS        00000000004006d0 0006d0 0000f4 00   A  0   0  8
  [18] .init_array       INIT_ARRAY      0000000000600e10 000e10 000008 00  WA  0   0  8
  [19] .fini_array       FINI_ARRAY      0000000000600e18 000e18 000008 00  WA  0   0  8
  [20] .jcr              PROGBITS        0000000000600e20 000e20 000008 00  WA  0   0  8
  [21] .dynamic          DYNAMIC         0000000000600e28 000e28 0001d0 10  WA  6   0  8
  [22] .got              PROGBITS        0000000000600ff8 000ff8 000008 08  WA  0   0  8
  [23] .got.plt          PROGBITS        0000000000601000 001000 000040 08  WA  0   0  8
  [24] .data             PROGBITS        0000000000601040 001040 000010 00  WA  0   0  8
  [25] .bss              NOBITS          0000000000601050 001050 000008 00  WA  0   0  1
  [26] .comment          PROGBITS        0000000000000000 001050 000024 01  MS  0   0  1
  [27] .shstrtab         STRTAB          0000000000000000 001074 0000f8 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
</pre>

可以看到，这个ELF文件里有28个sections。经常提到的比如`.text`, `.bss`, `.got`。


然后来看segments:
<pre>
lrk@laptop:~/tmp/CTF/ISG/pwnme$ readelf -Wl pwnme 

Elf file type is EXEC (Executable file)
Entry point 0x4004d0
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0001f8 0x0001f8 R E 0x8
  INTERP         0x000238 0x0000000000400238 0x0000000000400238 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x0007c4 0x0007c4 R E 0x200000
  LOAD           0x000e10 0x0000000000600e10 0x0000000000600e10 0x000240 0x000248 RW  0x200000
  DYNAMIC        0x000e28 0x0000000000600e28 0x0000000000600e28 0x0001d0 0x0001d0 RW  0x8
  NOTE           0x000254 0x0000000000400254 0x0000000000400254 0x000044 0x000044 R   0x4
  GNU_EH_FRAME   0x000698 0x0000000000400698 0x0000000000400698 0x000034 0x000034 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x000e10 0x0000000000600e10 0x0000000000600e10 0x0001f0 0x0001f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .jcr .dynamic .got
</pre>

看到这里有9个segments。只得注意的是`GNU_STACK`是RW，没有E，说明栈是不可执行的。

似乎sections的名字都是小写，segments的名字都是大写。

通过这些具体输出，再回顾下前面所说的3条：

1. 每个segments可以包含多个sections。这可以从"Section to Segment mapping"这里看到，比如说，segment 08是`GNU_RELRO`，包含了`.init_array`, `.fini_array`, `.jcr`, `.dynamic`, `.got`这5个sections
2. 每个sections可以属于多个segments。比如说，`.got`既在segment 03(`LOAD`)里，也在segment 08(`GNU_RELRO`)里
3. segments之间可以有重合的部分。这可以从第2点推出的。或者我们直接看segments的地址，还以segment 03和08为例，第二个`LOAD`(segment 03)的地址是`0x600e10`, 大小是0x240；而`GNU_RELRO`的地址也是`0x600e10`，不过大小只有0x1f0。如果我们结合着前面sections的地址来看，会更清楚。


对于exploit来说，会关心程序运行时哪些内存地址是可写、可执行的。而

> the segments contain information needed at runtime, while the sections contain information needed during linking

因此应该多关注segments的信息。

以上面的文件为例，我们看下程序执行时的memory maps:
<pre>
gdb-peda$ vmmap
Start              End                Perm      Name
0x00400000         0x00401000         r-xp      /home/lrk/tmp/CTF/ISG/pwnme/pwnme
0x00600000         0x00601000         r--p      /home/lrk/tmp/CTF/ISG/pwnme/pwnme
0x00601000         0x00602000         rw-p      /home/lrk/tmp/CTF/ISG/pwnme/pwnme
0x00007ffff7a38000 0x00007ffff7bd1000 r-xp      /usr/lib/libc-2.20.so
0x00007ffff7bd1000 0x00007ffff7dd1000 ---p      /usr/lib/libc-2.20.so
0x00007ffff7dd1000 0x00007ffff7dd5000 r--p      /usr/lib/libc-2.20.so
0x00007ffff7dd5000 0x00007ffff7dd7000 rw-p      /usr/lib/libc-2.20.so
0x00007ffff7dd7000 0x00007ffff7ddb000 rw-p      mapped
0x00007ffff7ddb000 0x00007ffff7dfd000 r-xp      /usr/lib/ld-2.20.so
0x00007ffff7fcf000 0x00007ffff7fd2000 rw-p      mapped
0x00007ffff7ff8000 0x00007ffff7ffa000 r--p      [vvar]
0x00007ffff7ffa000 0x00007ffff7ffc000 r-xp      [vdso]
0x00007ffff7ffc000 0x00007ffff7ffd000 r--p      /usr/lib/ld-2.20.so
0x00007ffff7ffd000 0x00007ffff7ffe000 rw-p      /usr/lib/ld-2.20.so
0x00007ffff7ffe000 0x00007ffff7fff000 rw-p      mapped
0x00007ffffffde000 0x00007ffffffff000 rw-p      [stack]
0xffffffffff600000 0xffffffffff601000 r-xp      [vsyscall]
</pre>

第一个`LOAD`从`0x400000`开始，长度是0x7c4，属性是`R E`。对应在mem maps中，确实0x400000的那一个page是r-x。注意这里segment的大小是0x7c4，比一个page的大小0x1000要小；所以`0x400000`开始的整个page都是r-x

第二个`LOAD`从`0x600e10`开始，长度是0x240，到`0x600e10+0x240=0x601050`。而这样的话这一段segments就跨过了`0x600000`和`0x601000`两个page。实际在mem maps中，我们看到`0x600000`那个page是r--的，而`0x601000`那个page是rw-的。这似乎与第二个`LOAD`是`RW `有点变化。

实际上，我们再看`GNU_RELRO`那个segment，它也是从`0x600e10`开始，长度0x1f0，正好在`0x600000`那个page里。而`GNU_RELRO`这个segment的属性是`R`，所以`0x600000`那个page也就成了r--了；而下一page, `0x601000`，还是rw-的

另外，我们再看看那两个segments所包含的sections，发现`.init_array`, `.fini_array`, `.jcr`, `.dynamic`, `.got`这5个sections虽然在第二个`LOAD`里，但他们也是在`GNU_RELRO`里，所以不可写的；剩下的`.got.plt`, `.data`, `.bss`则是可写的。之前用的改库函数地址，实际上就是改`.got.plt`。
