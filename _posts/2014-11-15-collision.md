---
title: collision
author: rk700
layout: post
redirect_from: 
  - /writeup/2014/11/15/collision
  - /writeup/2014/11/15/collision/
tags:
  - linux
  - pwnable.kr
---
代码如下

{% highlight c %}
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}

{% endhighlight %}

从题意来看，输入5个int共20bytes，其和要求为`0x21DD09EC`。于是可以让前4个数为`0xffffffff=-1`，最后一个数为`0x21dd09ec+4=0x21dd09f0`

具体在输入时，用的是`perl`。一开始没有用双引号，由于`0x09`是`\t`，被shell认为是参数分隔了。于是用双引号括起来：

<pre>$ ./col "$(perl -e 'print "\xff"x16 . "\xf0\x09\xdd\x21"')"</pre>

