---
title: 'Live LFI&#038;RFI'
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/06/18/live-lfirfi
  - /writeup/2014/06/18/live-lfirfi/
tags:
  - PHP
  - wechall
---
<http://www.wechall.net/challenge/warchall/live_rfi/index.php>  
<http://www.wechall.net/challenge/warchall/live_lfi/index.php>  
这两题类似，都是文件包含，参数lang可以传文件名

LFI那里，直接访问solution.php，有两行:teh falg si naer, the flag is near。一开始我还以为flag在另一个文件里，因为看论坛上说是在s\***\*\\*\*n.\*\**里。我以为提示是说重排文件名的字符，最后没成功……

然后发现我们可以ssh到warchall上，于是新建一个文件`/tmp/1`，里头是PHP代码，读solution.php的内容。再LFI这个文件就可以看到答案了

RFI与此类似，不过提示说有防火墙，只允许连几个地址。我们发现warchall也向用户提供web服务，于是开启之(`/home/feature/webserver_on`)，同样的办法，读取solution.php的内容，得到flag

另外，看论坛里其他人的解答，RFI不需要设web服务，可以用data协议：  
<pre>http://rfi.warchall.net/index.php?lang=data://text/plain,&#x3C;?php echo &#x60;your code&#x60;;?&#x3E;</pre>
