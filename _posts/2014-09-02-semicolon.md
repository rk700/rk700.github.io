---
title: 过滤了分号
author: rk700
layout: post
tags:
  - XSS
---
过滤了分号后，不能结束语句了，但可以通过二元运算符来。比如  
`var a = 1 + alert(2)`  
加号也可以换成减号、乘、除、并、或、异或