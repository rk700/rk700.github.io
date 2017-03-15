---
title: Leviathan
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/07/06/leviathan
  - /writeup/2014/07/06/leviathan/
tags:
  - OverTheWire
---
*  **leviathan0**  
    家目录下有个.backup文件夹，里面有一个bookmarks.html。直接用grep搜leviathan，得到密码
*  **leviathan1**以其为结尾  
    反编译，发现只是简单地比较字符串，得到密码
*  **leviathan2**  
    没发现溢出，单纯race condition的话又有点来不及，所以是command injection. 
    在当前目录下创建一个到目标文件的链接passwd->leviathan3，再创建一个文件名为`anyfile;cat passwd`的文件。那么在`system`执行的时候，命令注入，目标文件的内容会通过链接passwd获得
*   **leviathan3**  
    是在do_stuff函数里，简单的字符串比较
*   **leviathan4**  
    他会把密码的每个字符编码，然后输出。所以我们可以先构造码表，然后反推出密码。下面是python代码，其中字母表包括换行符，因为文件中可能以其为结尾

{% highlight python %}
#!/usr/bin/env python2
import sys

def convert(c):
    res = []
    n = ord(c)
    i = 0
    while i <= 7:
        if n & 128: #negative
            res.append('1')
        else:
            res.append('0')
        i = i+1
        n = (n<<1) & 255
    return ''.join(res)

if __name__ == '__main__':
    alpha = '0123456789qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM\x0a'
    book = [convert(x) for x in alpha]

    codes = sys.argv[1].split(' ')

    res = []
    for code in codes:
        idx = book.index(code)
        res.append(alpha[idx])

    print ''.join(res)
{% endhighlight %}



*   **leviathan5**  
    程序会输出某文件的内容，于是把目标文件链接到那里即可
*   **leviathan6**  
    简单地比较数字
