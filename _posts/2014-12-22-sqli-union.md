---
title: webhacking.kr challenge 7
author: rk700
layout: post
redirect_from: /writeup/2014/12/22/sqli-union
tags:
  - webhacking.kr
  - sql
---

发现有页面的[源代码](http://webhacking.kr/challenge/web/web-07/index.phps)，发现参数`val`处存在注入，而且在提示里也说了用`union`。所做的防御是过滤了一些字符，而且每次随机选一个查询语句，不同的语句需要闭合的括号数目不同。

由于不允许使用空格和注释，我就用`TAB`来分词；由于最后需要的结果2也不允许出现，加号也不允许出现，我们用减法`3-1=2`。

由于只有5种随机的情况，所以选定一种一直跑应该就有选中的可能。我选了最简单的只有1个括号的，

<pre>http://webhacking.kr/challenge/web/web-07/index.php?val=0)union%09select(3-1</pre>

没想到第一次跑就中了。
