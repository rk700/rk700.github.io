---
title: 'Training: WWW-Rewrites'
author: rk700
layout: post
redirect_from: /writeup/2014/07/21/training-www-rewrites
tags:
  - wechall
  - HTTP
---
[http://www.wechall.net/challenge/training/www/rewrite/index.php][1]  
比www-basic复杂的是，我们要url rewrite。这需要在httpd.conf里去掉rewrite_module的注释，并且加上  
<pre>
RewriteEngine  on
RewriteRule ^/nabla/([0-9]+)_mul_([0-9]+).html /nabla/1.php?v1=$1&#038;v2=$2
</pre>
然后我们在1.php里做乘法。

开始是简单地将两数相乘，但结果和正确的有差别。发现数字很大，所以用大数乘法得到正确答案 

{% highlight php %}
<?php
$v1 = $_GET['v1'];
$v2 = $_GET['v2'];
$mul=gmp_mul($v1,$v2);

echo gmp_strval($mul);
?>
{% endhighlight %}

注意还要将php.ini里的gmp.so的注释去掉，然后重启httpd

 [1]: http://www.wechall.net/challenge/training/www/rewrite/index.php "http://www.wechall.net/challenge/training/www/rewrite/index.php"