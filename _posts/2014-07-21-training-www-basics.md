---
title: 'Training: WWW-Basics'
author: rk700
layout: post
redirect_from: /writeup/2014/07/21/training-www-basics
tags:
  - wechall
  - HTTP
---
[http://www.wechall.net/challenge/training/www/basic/index.php][1]  
由于我的机子在内网，所以要先在路由器里设虚拟服务器。ip地址设为我的机子在内网的地址，端口80。然后把要求的文件放到对应地方即可

此外，在httpd.conf里，要把监听的设好，`Listen 80`

 [1]: http://www.wechall.net/challenge/training/www/basic/index.php "http://www.wechall.net/challenge/training/www/basic/index.php"