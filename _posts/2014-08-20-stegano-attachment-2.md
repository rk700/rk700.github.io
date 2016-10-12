---
title: stegano attachment
author: rk700
layout: post
tags:
  - wechall
---
<a title="http://www.wechall.net/challenge/training/stegano/attachment/index.php" href="http://www.wechall.net/challenge/training/stegano/attachment/index.php" target="_blank">http://www.wechall.net/challenge/training/stegano/attachment/index.php</a>  
下载图片，用Stegsolve打开，发现有additional bytes，共136。于是  
`tail -c 136 attachment.jpe > 1.zip`  
解压后得到密码

但是后来看论坛，直接解压图片都可以。因为

> zips store all required information at the end of the file and a typical unzipper basically jumps to the end and follows the pointers from there.