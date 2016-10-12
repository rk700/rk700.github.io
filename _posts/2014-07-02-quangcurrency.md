---
title: Quangcurrency
author: rk700
layout: post
tags:
  - wechall
---
<http://www.wechall.net/challenge/quangcurrency/index.php>  
论坛里有提示，说是标题就是hint，从这个标题可以想到concurrency，虽然我是答完后看论坛才知道

具体的，click会将金额加1，也就是new = old + 1, set value = new；不自然的地方就在于，相比其他操作，click花的时间要长的多的多。所以，如果在click运行中buy，就有concurrency的隐患，也就是买完后click才结束，并将金额重置回之前的加1

于是我们先点click的链接，然后立即点buy的链接，这样就会引起并发的问题