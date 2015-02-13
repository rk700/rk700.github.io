---
title: wireshark, burpsuite与SSL
author: rk700
layout: post
categories:
  - article
tags:
  - network
  - crypto
---

最近在工作中，有时需要用wireshark抓包分析。而有一些通信是被ssl加密了的。虽然说以前在其他队伍的writeup里有见到过如何把server的私钥提供给wireshark来解密，但我这几次试了下还是得不到需要的东西，所以稍微研究了一下。最后的结论是某些cipher是无法仅仅从抓到的包解密的。

首先，来了解一些相关的密码学知识。RSA就不再赘述了，这里主要讲一下Diffie-Hellman(DH)。从课堂上所学到的，知道通过它，可以在一个公开可监听的网络中，双方协商确定一个秘密，从而作为之后的密钥进行加密通信。如下是大致的描述：

* Alice和Bob首先确定一个素数p，以及p的原根g。这里p和g是可以被攻击者知道的
* Alice选择一个秘密的数a，计算A=g^a mod p，将结果传给Bob
* Bob选择一个秘密的数b, 计算B=g^b mod p, 将结果传给Alice
* Alice计算B^a mod p，作为他们的共享秘密
* Bob计算A^b mod p，作为他们的共享秘密

可以看到，最后二人计算得到的共享秘密是相同的，因为(g^b)^a=(g^a)^b。而攻击者知道的是p, g, A, B，但很难计算得到Alice和Bob的共享秘密（因为要计算离散对数）。同样因为离散对数，Alice几乎不可能知道Bob的秘密b，Bob几乎不可能知道Alice的秘密a。
更进一步，在得到最后的共享秘密后，a和b完全可以舍弃。这样也避免了a或b被他人得到而解密。一般来说，a和b都是生成的一次性的随机数，不会被保存。

（题外话：DH真的是非常漂亮的算法啊）

然后，我们来看看ssl加密。SSL/TLS是在transport layer之上。主要流程是：首先通过PKI验证身份并确定之后的共享密钥，然后用这个密钥进行对称加密，进行通信。毕竟非对称加密相比对称加密还是非常慢的。

下图是RFC5246里对ssl流程的描述：

<pre>
      Client                                               Server

      ClientHello                  -------->
                                                      ServerHello
                                                     Certificate*
                                               ServerKeyExchange*
                                              CertificateRequest*
                                   <--------      ServerHelloDone
      Certificate*
      ClientKeyExchange
      CertificateVerify*
      [ChangeCipherSpec]
      Finished                     -------->
                                               [ChangeCipherSpec]
                                   <--------             Finished
      Application Data             <------->     Application Data
</pre>

其中星号说明该项是可选项。

下面结合我抓到的包来具体地看流程中各个步骤。server的私钥已经导入到wireshark。client的地址是10.66.128.8，server监听10.66.79.150:8443。

首先是ClientHello，会告诉server一些信息，比如client支持的cipher suites, compression method等等。

![Clienthello]({{ site.url }}/assets/img/wireshark-burp-ssl/clientHello.png){:.myImage}

server会选取可接受的cipher和compression，由ServerHello告诉client。而我抓到的包里，cipher用的是TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256，其中ECDHE的含义是elliptical curve Diffie-Hellman ephemeral。而这个cipher导致了我最终没能解开加密，后面会详细说。

![ServerHello]({{ site.url }}/assets/img/wireshark-burp-ssl/serverHello.png){:.myImage}

发送完ServerHello，server会把自己的证书发给client，由client检查是否可信。wireshark里可以看到证书具体每项的内容。

![server的证书]({{ site.url }}/assets/img/wireshark-burp-ssl/serverCert.png){:.myImage}

此外，server也可以选择检查client的身份，通过发送CertificateRequest并检查client提供的证书。

然后来看ServerKeyExchange和ClientKeyExchange。根据介绍

> The ServerKeyExchange message is sent by the server only when the server Certificate message (if sent) does not contain enough data to allow the client to exchange a premaster secret.

而对于我们这里的ephemeral DH，就属于这种情况。(ephemeral RSA也一样)。也就是说，server证书里的RSA密钥并不用来传递共享密钥，共享密钥是通过前面所说的ECDHE传递的，RSA密钥的作用仅仅是为ECDHE做签名。这样临时的ephemeral DH key是一次性的，相对来说更安全一些。RFC4346里这么说：

> When Diffie-Hellman key exchange is used, the server can either supply a certificate containing fixed Diffie-Hellman parameters or use the server key exchange message to send a set of temporary Diffie-Hellman parameters signed with a DSS or RSA certificate.  Temporary parameters are hashed with the hello.random values before signing to ensure that attackers do not replay old parameters. In either case, the client can verify the certificate or signature to ensure that the parameters belong to the server.  

再回顾DH，我们知道，即使通过server的私钥解密了DH的过程，我们还是无法计算出DH所确定的共享密钥，因此后面的对称加密也就解密不了了。这就解释了为什么提供server的私钥也不能解密。

**补记：**

在[这里](https://jimshaver.net/2015/02/11/decrypting-tls-browser-traffic-with-wireshark-the-easy-way/)提到，如果是要解密firefox的通信，可以通过环境变量`SSLKEYLOGFILE`，这样就可以[将master key保存下来](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/Key_Log_Format)，进而wireshark可以由master key直接解密。但这个只是针对于Network Security Services(NSS)。

如果SSL用的不是NSS而是其他诸如OpenSSL呢？[ssltrace](https://github.com/jethrogb/ssltrace)似乎是个不错的选择。根据其描述，它是通过`LD_PRELOAD`来hook一些函数，支持OpenSSL和GnuTLS。但对于我还是用不了，因为我的二进制文件是Go编译的静态链接文件，不引用OpenSSL，悲催……

于是，用wireshark解密似乎是不太好办了，于是另一个思路就是用burp代理截包。burp监听本地的8080端口，但我的二进制文件不支持设置代理，于是可以通过iptables端口转发来将我们的request转到burp那里去。而在这里有一个坑：我把server放到虚拟机里，地址是127.0.0.1:8443，通信走的是本地lo。而我最初将iptables如下设置：

<pre>$ iptables -A OUTPUT -t nat -p tcp --dport 8443 -j REDIRECT --to-ports 8080</pre>

再启动client访问server，发现burp里开始无限循环访问server。这是因为burp抓到我们的包再去访问127.0.0.1:8443时，iptables规则再次生效，又一次发给8080的burp，如此无限循环了……在网上搜索，发现[解决方法](http://serverfault.com/questions/568656/iptables-causes-infinite-loop-of-get-request)，即排除iptables规则对某一用户的程序的适用：

<pre>$ iptables -A OUTPUT -t nat -p tcp --dport 8443 -m owner ! --uid-owner 0 -j REDIRECT --to-ports 8080</pre>

这里uid为0的用户即root。所以我们只需要以root身份启动burp就可以了。

到这里还没完……接下来是burp的坑了。我最初用burp的默认配置，结果抓不到包，然后在设置里勾选上了support invisible proxy后才可以。但紧接着tls报错: failed to parse certificate from server: x509: negative serial number。检查后发现确实burp签名的证书serial number是负数。

![证书serial number是负数]({{ site.url }}/assets/img/wireshark-burp-ssl/negSN.png){:.myImage}

于是继续在网上搜索，看到[burp论坛](http://forum.portswigger.net/thread/1503/burp-cert-serial-numbers-compliant)里提到过这件事。不幸的是Go TLS不支持负的serial number。那篇帖子的时间是2014年10月，我用的burp是free edition 1.6，可能还是需要等burp修复这个问题了。

