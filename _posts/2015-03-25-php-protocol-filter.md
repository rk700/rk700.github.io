---
title: PHP protocol的坑
author: rk700
layout: post
redirect_from: /article/2015/03/25/php-protocol-filter
tags:
  - PHP
---

LFI(local file inclusion)是一类比较典型的漏洞，不过之前也并没怎么利用过它，主要还是`include`时往往有许多限制，比如文件的后缀必须是`.php`。今天偶然发现了PHP伪协议的一个坑，可以得到php文件的内容而不是作为php解释。

这是`php://filter/convert.base64_encode/resource=`。根据官方文档的[描述](http://php.net/manual/en/filters.convert.php)，他可以像`base64_encode()`那样将stream进行编码。于是，在如下的场景中

{% highlight php %}
<?php 

include($_GET["var"] . ".php"); 

?>
{% endhighlight %}

如果我们访问`index.php?var=php://filter/convert.base64-encode/resource=index`，那么就会得到`include(php://filter/convert.base64-encode/resource=index.php)`，将`index.php`的内容进行base64编码了。由此，我们可以解码得到`index.php`的代码。

这种方法还有一点好处，是不包含截断用的`\x00`字符。当然，局限就是读后缀被对方限制的文件。不过能获取源代码，也是比较危险的了。

话说回来，之前ISG初赛有一道题目就是利用了PHP的`data://text/plain;base64,`这种protocol来绕过过滤。确实这些protocol给程序员带来了不少方便，但也埋下了一个个坑，给黑名单过滤这种机制带来隐患。
