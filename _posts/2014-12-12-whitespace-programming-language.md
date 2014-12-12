---
title: Whitespace programming language
author: rk700
layout: post
categories:
  - article
tags:
  - misc
---

今天做题遇到了一道很奇怪的题目。他给了一个[cpp文件](http://ksnctf.sweetduet.info/q/7/program.cpp)

单看代码，似乎只是把普通的c++代码添加里很多tab，空格，并没有什么奇怪的地方。用g++编译后，执行得到说`FROG_This_is_wrong_:(`。看来是另有玄机。

后来搜里下，找到了一种奇葩的编程语言，[whitespace](http://en.wikipedia.org/wiki/Whitespace_%28programming_language%29)。具体在[这里](http://compsoc.dur.ac.uk/whitespace/tutorial.php)有更详细的介绍。简单地说，它是只考虑空白字符，通过特定的组合对应特定的指令。

然后找这个语言的编译器，发现里[这里](http://mearie.org/projects/esotope/ws)有用python实现的一个。更奇葩的是，它本身的python脚本是还是组成了一个人物头像，这帮人怎么就这么爱玩啊……一直以为python缩进管的很严格的，这次真是见到奇葩了

具体地，用`python esotope-ws -d program.cpp program.wsa`得到“汇编”代码。其内容如下（我添加了一些注释。由于不知到用什么注释符号，就用//了）

<pre>
  push 80 //P
  putchar
  push 73 //I
  putchar
  push 78 //N
  putchar
  push 58 //:
  putchar
  push 32 //space
  putchar
  push 0 
  getint //store the input int at position '0'
  push 0
  retrieve //put content at position '0' to top of the stack
  push 33355524
  sub
  jz label1_1 //input-33355524
  push 78 //N
  putchar
  push 79 //O
  putchar
  halt
label1_1:
  push 79 //O
  putchar
  push 75 //K
  putchar
  push 10 //\n
  putchar
  push 0
  retrieve //input must be 33355524
  push 33355454
  sub
  dup
  putchar //F
  push 6 
  add
  dup
  putchar //L
  push 11
  sub
  dup
  putchar //A
  push 6
  add
  dup
  putchar //G
  push 24
  add
  dup
  putchar //underscore
  push 26
  sub
  dup
  putchar //E
  push 40
  add
  dup
  putchar
  push 25
  sub
  dup
  putchar
  push 36
  add
  dup
  putchar
  push 66
  sub
  dup
  putchar
  push 16
  add
  dup
  putchar
  push 14
  add
  dup
  putchar
  push 14
  add
  dup
  putchar
  push 27
  sub
  dup
  putchar
  push 5
  add
  dup
  putchar
  push 29
  add
  dup
  putchar
  push 4
  sub
  dup
  putchar
  push 4
  add
  dup
  putchar
  push 28
  sub
  dup
  putchar
  push 22
  add
  dup
  putchar
  push 34
  sub
  dup
  putchar
  push 55
  sub
  dup
  putchar
  halt
</pre>
  
  由于二元运算后，两个operand都会从栈上弹出，把结果放栈顶，然后putchar又会用掉，于是有好几处用dup复制了一份，以便下面的计算。我们可以看到，如果输入的数等于33355524，就会把flag打印出来。我计算了前几个输出字符，确实是"FLAG_"

  然后后面的就没有算了，想找一个编译器，也没有在本地搭，而是有个在线的[网站](http://ideone.com)支持whitespace。把program.cpp复制到代码里，在把33355524放到stdin，运行得到flag
