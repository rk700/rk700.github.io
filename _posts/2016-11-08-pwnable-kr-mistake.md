---
title: pwnable.kr之mistake
author: rk700
layout: post
redirect_from: 
tags:
  - linux
  - pwnable.kr
---

由这道题目的提示可知，坑是在运算符优先级中。具体地，在C中，比较运算符（如`>`, `<`）的优先级，比赋值运算符（`=`）是高的。

而具体地，检查提供的代码，在打开文件时便存在问题：

{% highlight c %}
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
    printf("can't open password %d\n", fd);
    return 0;
}
{% endhighlight %}

此处，虽然`=`两边没有空格，`<`两边有空格，使人觉得这里的运算顺序是：

1. 调用`open`，并将结果保存在变量`fd`中
2. 检查`fd`是否小于0

但实际上，由于之前提到的运算符优先级，这里实际的运算顺序是：

1. 调用`open`，并检查其返回结果是否小于0
2. 将上一步结果保存在变量`fd`中

如果直接查看对应的汇编代码，应该会更清晰一些：

{% highlight nasm %}
mov     edi, offset file ; "/home/mistake/password"
mov     eax, 0
call    _open
shr     eax, 1Fh
mov     [rbp+fd], eax
cmp     [rbp+fd], 0
jz      short loc_4008B5
mov     eax, offset format ; "can't open password %d\n"
{% endhighlight %}

所以，在写C代码时，如果对运算符优先级不确定，最好还是多使用圆括号来指定运算顺序，否则就可能像这里一样犯mistake了。

回到题目本身。调用`open`的返回值是大于0的，按照实际的运算顺序，其比较结果为false，即`fd=0`。这里`=`赋值运算的返回值，仍然是所赋的值0，所以`if`判断不通过，继续进行下面的操作。

而0所对应的是标准输入，所以接下来便是从标准输入中读入内容，作为`pw_buf`的内容，并将其与随后进行XOR的输入内容比较。

进行XOR的key值是1，我们便选取两个字符`B`和`C`，因为`ord('B')^ord('C') == 1`。由此，便可以成功读取flag。

