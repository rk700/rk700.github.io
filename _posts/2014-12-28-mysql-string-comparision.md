---
title: mysql比较字符串忽略结尾的空白
author: rk700
layout: post
categories:
  - article
tags:
  - sql
---

今天学习到一个知识点，在查询时如果比较字符串，会忽略结尾的连续空白，起始的空白不会忽略。于是：

<pre>select * from users where user='admin ';</pre>

会把`user='admin'`的记录也选出。

[这里](http://stackoverflow.com/questions/7455147/comparing-strings-with-one-having-empty-spaces-before-while-the-other-does-not)有比较详细的例子，而且mysql的[官方文档](http://dev.mysql.com/doc/refman/5.0/en/string-comparison-functions.html)也说了：

> In particular, trailing spaces are significant, which is not true for CHAR or VARCHAR comparisons performed with the = operator 
