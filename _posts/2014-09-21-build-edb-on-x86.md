---
title: build edb on x86
author: rk700
layout: post
categories:
  - article
tags:
  - linux
  - secTool
---
我想在32位的环境下用edb，按理来说是支持的，但编译时发现他认为是在64位下面……可能还是和chroot有关。在网上搜了半天也没找到什么好的结果，最后用的是笨办法，把src/src.pro和plugins/plugins.pri里面头文件路径都用成x86，再编译就可以了