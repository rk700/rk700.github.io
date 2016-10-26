---
title: webhacking.kr challenge 8
author: rk700
layout: post
redirect_from: /writeup/2014/12/22/injection-in-useragent
tags:
  - webhacking.kr
  - sql
---

接连的几道题都是代码审计的。[这道题](http://webhacking.kr/challenge/web/web-08/index.phps)里，对`User-Agent`有两个取法，一种是`getenv`，另一种是`$_SERVER`，而对两个的不一致的过滤就造成了问题。


具体地，`getenv`得到的里面不限制单引号、括号和注释符号#，而且会在后面的插入语句里出现。如果我们这里取`User-Agent: agent','ip', 'admin')#`，那么就会插入一条id为admin的记录。

接下来，我们再请求一次，这次是`User-Agent: agent`，那么就会取出刚才插入的那条记录，满足id为admin的条件。
