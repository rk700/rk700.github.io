---
title: 0CTF rsa quine
author: rk700
layout: post
redirect_from: /writeup/2015/04/03/0ctf-rsaquine
tags:
  - crypto
---

这道题是一道密码学（其实是数学）的题目，因为对相关理论还是不够熟练，在比赛的时候没有能够做出来:( 这两天读了读[别人的writeup](https://gist.github.com/mheistermann/0dee124d7eed2ec26fcd)，基本上明白解法了，在这里记录下具体的思路。

题目的要求是求"quine number"，即：已知RSA公钥$$(e, n)$$，求$$m$$，使得$$m^e\equiv m \bmod n$$。规则很清晰，但实际解决起来还是需要一定的数学知识。

Ok. Now welcome to the world of math!

我当时犯的一个错误是，误以为$$m^{e-1}\equiv 1\bmod n$$。满足这样的$$m$$当然符合条件，然而试了下就发现，所有这样的$$m$$还是不够题目要求的quine的数目。事实上，这样做的前提是存在$$m$$的逆$$m^{-1}$$，而$$n$$是一个合数，$$\mathbb{Z}_n$$是环而不是域，逆不一定存在的。

于是，我们首先分解$$n=pq$$。因为$$p$$与$$q$$互素，所以题目就可以化为：求$$m$$使得

$$m^e\equiv m\bmod p,\quad m^e\equiv m\bmod q$$

而$$p$$和$$q$$是素数，所以$$\mathbb{Z}_p$$和$$\mathbb{Z}_q$$就是有限域了，必然存在逆。所以，如果记$$\hat{m}=m\bmod p$$，$$\bar{m}=m\bmod q$$，题目可以进一步转化为：求$$\hat{m}$$，$$\bar{m}$$使得

$$\hat{m}^{e-1}\equiv 1\bmod p,\quad \bar{m}^{e-1}\equiv 1\bmod q$$

一旦得到了$$\hat{m}$$和$$\bar{m}$$，就可以根据中国剩余定理得到$$m$$。

下面来看怎样求$$\hat{m}$$。$$\mathbb{Z}_p$$存在作为乘法群的generator的primitive element，记为$$g$$。它具有如下性质：

$$g^{p-1}\equiv 1\bmod p;\quad\text{而且}~\forall~1\le y\le p-1,\; \exists~1\le x\le p-1\;s.t.\;g^{x}\equiv y\bmod p$$

当然$$\hat{m}$$可以为0；如果其非0，那么就可以被$$g$$生成。假设$$g^x\equiv \hat{m}\bmod p$$，那么$$g^{x(e-1)}\equiv 1\bmod p$$，从而$$x(e-1)$$一定是$$p-1$$的倍数。因此，我们有：

$$x(e-1)=k(p-1)$$

为了求所有的$$x$$，我们只需要先求出$$gcd(e-1,p-1)$$，那么$$x=\frac{p-1}{gcd(e-1,p-1)},\;\frac{2(p-1)}{gcd(e-1,p-1)},\;\frac{3(p-1)}{gcd(e-1,p-1)},\ldots$$。有了这些$$x$$，我们可以计算出相应的一系列$$\hat{m}$$。注意这里是有限域，所以所有的$$\hat{m}$$，除了0，其实也构成了一个cyclic group。

上述的就是如何求$$\hat{m}$$。同样的道理我们可以求出所有的$$\bar{m}$$。两两组合，每一对使用中国剩余定理，就得到了满足题意的$$m$$。

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
