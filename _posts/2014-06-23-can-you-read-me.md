---
title: Can you read me
author: rk700
layout: post
tags:
  - wechall
---
[http://www.wechall.net/challenge/can\_you\_readme/index.php][1]  
这道题试了下没发现什么漏洞，所以只好老老实实地去做识别了

搜了下有tesseract这个软件可以做ocr，于是用下面的脚本：

{% highlight bash %}
#!/bin/bash

wget -O ocr.png --no-cookies --header "Cookie:WC=7390992-11403-6nnwUyENAALTe4VK" \
http://www.wechall.net/challenge/can_you_readme/gimme.php

tesseract ocr.png ocr -psm 7

SOLUTION=`head -1 ocr.txt`

curl -b "WC=7390992-11403-6nnwUyENAALTe4VK" -G -d "solution=$SOLUTION" -d "cmd=Answer" \
http://www.wechall.net/challenge/can_you_readme/index.php
{% endhighlight %}

 [1]: http://www.wechall.net/challenge/can_you_readme/index.php