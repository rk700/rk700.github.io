---
title: Crappyshare
author: rk700
layout: post
categories:
  - writeup
tags:
  - wechall
  - PHP
---
<http://www.wechall.net/challenge/crappyshare/index.php>  
检查代码，发现如果是通过url，会将文件内容也输出来。为了读取本地文件，我们用`file://`协议  
`file://solution.php`