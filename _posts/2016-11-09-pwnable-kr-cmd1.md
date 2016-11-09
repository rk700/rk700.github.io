---
title: pwnable.kr之cmd1
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - pwnable.kr
---

由这道题目的提示可知，应该与环境变量有关。具体地，程序首先通过`putenv`将`PATH`设置为一个不存在的地址:

{% highlight c %}
putenv("PATH=/fuckyouverymuch");
{% endhighlight %}

并检查了要执行的命令中是否包含了敏感字符串。

{% highlight c %}
r += strstr(cmd, "flag")!=0;
r += strstr(cmd, "sh")!=0;
r += strstr(cmd, "tmp")!=0;
{% endhighlight %}

由于`PATH`是一个不存在的地址，要执行命令必须提供绝对路径；另一方面，提供的命令中不能包含`tmp`，所以为了便于操作，我们可以通过环境变量来达到目的。例如，我们首先进入`/tmp`目录再离开，则此时的`OLDPWD=/tmp`。

由此，通过环境变量来代替敏感字符串`tmp`，便可执行我们放在`/tmp`目录下的脚本了，从而读取flag文件。

