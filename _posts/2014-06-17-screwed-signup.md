---
title: Screwed Signup
author: rk700
layout: post
categories:
  - writeup
tags:
  - sql
  - wechall
---
<http://www.wechall.net/challenge/screwed_signup/index.php>  
这道题比较有意思

经过一番尝试，发现了代码中不一致的地方：SQL table里username最多24个字符，但是preg_match检查时可以最多到64个。于是这里可能造成截断……

另一个不协调的地方是`signupGetUser`。可以看到，在另一个函数里检查了密码，但取用户信息的这个函数只检查了用户名。所以，只要表中存在另外一个帐号Admin/password，那么我们以那个帐号登陆后，`signupGetUser`会取第一条出来，也就是把真正的管理员信息取出来了

于是我们利用截断来注册一个用户名也是Admin的帐号

> If strict SQL mode is not enabled and you assign a value to a CHAR or VARCHAR column that exceeds the column&#8217;s maximum length, the value is truncated to fit and a warning is generated.
> 
> For VARCHAR columns, trailing spaces in excess of the column length are truncated prior to insertion and a warning is generated, regardless of the SQL mode in use. 

当注册用户名为`Admin+大于20个空白+abc`时，结尾的abc保证了中间的空白不会被截断，然后preg_match也通过，判断用户是否存在也通过（因为不存在这样的用户名）。然后insert时会截断，而且空白也截断了，实质上造成了另一个用户名为Admin的帐号的添加。之后我们以此登陆，就ok了