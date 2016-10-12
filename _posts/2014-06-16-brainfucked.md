---
title: Brainfucked
author: rk700
layout: post
tags:
  - javascript
  - wechall
---
<https://www.wechall.net/challenge/brainfucked/index.php>

这道题是jother混淆的javascript。用jother可以编码成字符串或者是函数，如果是字符串那么直接在浏览器里运就可以。这里是函数，实际上是`f(something)()`的形式，那么只要把`something`运行就可以

或者将其保存下来，直接用`node`运，因为没有`document`等元素，会报错，这时我们可以看到其内容