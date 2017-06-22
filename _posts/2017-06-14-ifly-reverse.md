---
title: 讯飞输入法PC版日志上传解密逆向
author: rk700
layout: post
catalog: true
tags:
  - reverse
---

## 入手

驰那边安装讯飞输入法后，他将全部文件打包发送过来。解压后发现有`.exe`和`.dll`文件：

![]({{ site.url }}/img/in-posts/ifly-reverse/filelist.png){:.myImage}

所以要检查的目标，就是这些二进制文件。

根据旭抓包的结果，日志请求会访问 `/log.aspx?c=1002&v=2.0&t=20170613144351` 这样的地址，所以，最基本的就是在全部二进制文件中搜索字符串`&t=`。搜索确认后，发现在文件BlcCore.dll中存在这条格式化字符串。

而查找这条字符串的交叉引用，发现只被`blc::WebEngine::start(int **this, char *a2)`调用：

![]({{ site.url }}/img/in-posts/ifly-reverse/searchStrRes.png){:.myImage}

所以，这个函数就是我们接下来主要分析的目标。

## 时间戳

上文提到的格式化字符串，出现在这里的`sprintf()`调用：

![]({{ site.url }}/img/in-posts/ifly-reverse/strAppear.png){:.myImage}

因为我们从http请求，知道`t`的值是时间戳，所以`sprintf()`的最后一个参数应该就是字符串`20170613144351`。

## Buf结构体

反推时间戳的来源，发现是从一个自定义的结构体中得来的。而这个结构体是从方法`blc::WebEngine::getCurDate()`得到的：

![]({{ site.url }}/img/in-posts/ifly-reverse/getCurDate_impl.png){:.myImage}

这个结构体实现了动态buf管理，应该类似于C++中的`std::string`，结合经验和具体的代码，在IDA中新建一个对应的结构体类型`Bufstruct`：

![]({{ site.url }}/img/in-posts/ifly-reverse/bufstruct.png){:.myImage}

并将一些类似的变量类型都修改为`Bufstruct`

## http请求body

继续检查函数`blc::WebEngine::start(int **this, char *a2)`，发现在下方有调用`blc::HttpEngine::setRequestBody()`，这个方法的第一个参数是`this`指针，第二个是`char *`，第三个是`unsigned int`：

![]({{ site.url }}/img/in-posts/ifly-reverse/setBody_impl.png){:.myImage}

所以可猜测，第二个参数是请求body的内容，第三个参数是请求body的长度：

![]({{ site.url }}/img/in-posts/ifly-reverse/setBody_call.png){:.myImage}

而这里反推`v5`的来源，可以确定这是一个大小为8 bytes的结构体，保存了body的指针和长度：

![]({{ site.url }}/img/in-posts/ifly-reverse/requestBody.png){:.myImage}

函数`sub_100067E0()`就是生成这个结构的方法，可以再次确认这个结构保存的是长度和指针：

![]({{ site.url }}/img/in-posts/ifly-reverse/setBody_imp.png){:.myImage}

于是，我们接下来就只需要检查在调用`setRequestBody()`之前，对请求body结构体进行了哪些操作。

## 找到xor

在调用`setRequestBody()`之前，发现请求body的内容有被进行了一个xor操作：

![]({{ site.url }}/img/in-posts/ifly-reverse/xor.png){:.myImage}

由此可知，这里生成了某个xor密钥，以`Bufstruct`结构形式保存，并且将其与请求body做了正好一次异或加密（无循环）。所以，接下来就只需要确定这个`xorKey`是如何生成的

## 找到xor密钥生成方法

向上查找调用情况，发现xor密钥是通过某个拼接函数将两个`Bufstruct`拼接得到的：

![]({{ site.url }}/img/in-posts/ifly-reverse/callConcat.png){:.myImage}

这个拼接函数的实现如下：

![]({{ site.url }}/img/in-posts/ifly-reverse/concat_impl.png){:.myImage}

具体跟进函数`appendBufstruct()`，确认其是将第一个buf附加到第二个buf的后面：

![]({{ site.url }}/img/in-posts/ifly-reverse/concat1.png){:.myImage}

再根据这个方法调用的具体参数，可知`xorKey`是将请求body的长度附加到了时间戳的后面

![]({{ site.url }}/img/in-posts/ifly-reverse/concat2.png){:.myImage}


## 总结

由以上逆向分析可知，上传请求时，其query中的时间戳和请求Body的长度会拼接到一起，得到一个xor密钥。随后对请求body的起始部分，做一次异或加密，将结果作为最终的请求Body。所以，在实际逆向时，可构造同样的密钥，再次异或解密。解密的结果，就会很容易分析出来是gzip压缩包了。
