---
title: pwnable.kr之cmd2
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - pwnable.kr
---

这道题目与cmd1类似，也是设置了环境变量`PATH`，并检查输入的命令是否包含敏感字符串。所不同的是，这次把其他环境变量都清空了，所以需要通过别的方式来绕过。

具体地，检查敏感字符串如下：

{% highlight c %}
r += strstr(cmd, "=")!=0;
r += strstr(cmd, "PATH")!=0;
r += strstr(cmd, "export")!=0;
r += strstr(cmd, "/")!=0;
r += strstr(cmd, "`")!=0;
r += strstr(cmd, "flag")!=0;
{% endhighlight %}

这里，我们仍然要执行放在`/tmp`目录下的脚本，而且仍然要提供绝对路径。所以，我们通过shell的内置命令来完成。具体地，`eval`可将其后的字符串作为命令执行，我们接下来就需要构造目标字符串。

由于`PATH`的原因，我们依然使用内置命令来完成。具体地，我们使用`printf`，并且通过8进制的方式绕过对敏感字符的检查。本来是想直接使用16进制的方式，但不知为何，在目标环境上打印不出来。

为了得到目标字符串的8进制表示，我们使用如下命令：

{% highlight bash %}
$ echo -n '/tmp/ncmd2' | hexdump -v -e '"\\" 1/1 "%03o"'
{% endhighlight %}

随后，就可以通过`printf "\xxx"`这种方式得到目标字符串了。由于反引号也是敏感字符，所以我们使用`$()`的方式得到命令结果作为字符串。最终，调用`eval`执行

**Edit:** 后来发现，可以直接通过`$()`的方式将得到的字符串作为名称执行，不需要再使用`eval`。
