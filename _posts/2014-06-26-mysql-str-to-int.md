---
title: mysql字符串转为整数
author: rk700
layout: post
redirect_from: /article/2014/06/26/mysql-str-to-int
tags:
  - sql
---
<http://www.freebuf.com/articles/web/6894.html>

这里的原理是：&#8221;-&#8221;是字符串减字符串，而字符串会被转为0，就变成了0-0=0。然后再比较name和0，字符串name被转为0，得到0=0，条件成立

但这里条件是字符串name被转为0，如果name的第一个字符是数字而且不是0，那么就会被转成那个数字，此时条件不成立

另外，除了&#8221;-&#8221;，还可以&#8221;-0, &#8221;+0, &#8221;*0等等，只要使等号右边得到数字0就行