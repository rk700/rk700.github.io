---
title: 对加密的zip文件使用known plain text attack
author: rk700
layout: post
categories:
  - writeup
tags:
  - ksnctf
  - misc
---

[这道题](http://ksnctf.sweetduet.info/q/19/flag.zip)提示了说对zip文件使用known plain text attack，于是在网上搜了下，[这里](ftp://utopia.hacktic.nl/pub/crypto/cracking/pkzip.ps.gz)有一篇论文是关于这个的。

具体论文就没有看了，直接找了现成的实现[pkcrack](https://www.unix-ag.uni-kl.de/~conrad/krypto/pkcrack.html)。不过按照作者的要求，需要给他寄明信片，我没有寄诶……

按照要求，破解还需要知道一部分plain text。而加密的flag.zip文件里有一个`Standard-lock-key.jpg`文件，搜了在网上下找到了这个[文件](http://upload.wikimedia.org/wikipedia/commons/a/a2/Standard-lock-key.jpg)。

于是接下来就可以破解了。参考pkcrack的说明，我们运行

<pre>$ pkcrack -C ~/Downloads/flag.zip -c Standard-lock-key.jpg -p ~/Downloads/Standard-lock-key.jpg -d ~/1.zip</pre>

就得到没加密的压缩文件，解压后可得flag
