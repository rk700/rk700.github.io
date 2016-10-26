---
title: 'Impossible n&#8217;est pas français'
author: rk700
layout: post
redirect_from: /writeup/2014/07/02/impossible-nest-pas-francais
tags:
  - wechall
---
<http://www.wechall.net/challenge/impossible/index.php>  
这道题要求在6秒内分解一个超级大的整数，所以标题的名字都是&#8221;impossible&#8221;

这道题比较奇怪的一个地方是，如果你输入了错误的答案，他会把正确的答案告诉你，而这按理来说并不需要。于是从这里入手，发现即使答错，问题的整数还是没有变，这就给了我们再次尝试的机会。也就是说，先得到正确答案，然后在6秒之内再回答就行。

下面是python代码

{% highlight python %}
#!/usr/bin/env python2

import urllib
import urllib2
import re

if __name__ == "__main__":
    url = "http://www.wechall.net/challenge/impossible/index.php?request=new_number"
    req = urllib2.Request(url)
    req.add_header('cookie','WC=7436562-11403-u5AFYCNQVEgMepLw')
    response = urllib2.urlopen(req)

    url = "http://www.wechall.net/challenge/impossible/index.php"
    values = {'solution':1, 'cmd':'Send'}
    data = urllib.urlencode(values)
    req = urllib2.Request(url, data)
    req.add_header('cookie','WC=7436562-11403-u5AFYCNQVEgMepLw')
    response = urllib2.urlopen(req)
    content = response.read()
    
    p = re.compile('Correct would have been &quot;(\d*)&')
    sol = p.search(content).group(1)
    print sol

    values = {'solution':sol, 'cmd':'Send'}
    data = urllib.urlencode(values)
    req = urllib2.Request(url, data)
    req.add_header('cookie','WC=7436562-11403-u5AFYCNQVEgMepLw')
    response = urllib2.urlopen(req)
    content = response.read()
    print content
{% endhighlight %}