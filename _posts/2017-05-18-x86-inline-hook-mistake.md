---
title: x86架构下对方法做inline hook的坑
author: rk700
layout: post
catalog: true
tags:
  - binary
---

最近在研究Android原生代码hook时，遇到了一个麻烦。具体来说，就是在x86架构下，方法inline hook后，在执行原方法时可能会segfault。这里简要记录下，希望之后能够解决这个问题。

## inline hook的基本思路

对方法进行inline hook，基本上就是以下步骤：

- 将origin方法的起始几条指令，保存到backup
- 在backup的最后，跳转回origin方法接下来的指令
- 将origin方法的起始，修改为跳转到hook

我遇到的问题，就是调用backup时发生的。

## backup方法的坑

在进行了inline hook之后，如果需要调用origin方法，必须通过backup方法来完成：

- 调用backup
- 跳转回origin方法未被修改的地方

然而，即使backup的起始几条指令是从origin复制得来的，执行backup时就真的没有副作用吗？很不幸，这样想就太简单了！例如，如果复制到backup的指令中，包含了call/jmp等相对跳转指令，那么，在backup中，还需要对这些指令的目标偏移量做相应的调整。例如，在VirtualApp的MSHook中，有如下代码：

{% highlight c %}
...
if (backup[offset] == 0xe8) {
  int32_t relative(*reinterpret_cast<int32_t *>(backup + offset + 1));
  void *destiny(area + offset + decode.len + relative);

  if (relative == 0) {
      length -= decode.len;
      length += MSSizeOfPushPointer(destiny);
  } else {
      length += MSSizeOfSkip();
      length += MSSizeOfJump(destiny);
  }
} else if (backup[offset] == 0xeb) {
    length -= decode.len;
    length += MSSizeOfJump(area + offset + decode.len + *reinterpret_cast<int8_t *>(backup + offset + 1));
} else if (backup[offset] == 0xe9) {
    length -= decode.len;
    length += MSSizeOfJump(area + offset + decode.len + *reinterpret_cast<int32_t *>(backup + offset + 1));
...
{% endhighlight %}

然而，我所遇到的问题还并不止于此。

## x86架构下基于`pc`寄存器的内存访问

由于x86原生不支持相对于`pc`寄存器的内存访问，所以当出现这类需求时，需要首先通过某些手段，将`pc`寄存器复制到其他通用寄存器。例如：

{% highlight asm %}
.text:0002ACC7                 push    ebx
.text:0002ACC8                 sub     esp, 18h
.text:0002ACCB                 call    __x86_get_pc_thunk_bx
.text:0002ACD0                 add     ebx, 0CDA4Ch
{% endhighlight %}

这里的`__x86_get_pc_thunk_bx`，便是用于将`pc`复制到`ebx`：

{% highlight asm %}
.text:00016E3A __x86_get_pc_thunk_bx proc near 
.text:00016E3A                 mov     ebx, [esp+0]
.text:00016E3D                 retn
.text:00016E3D __x86_get_pc_thunk_bx endp
{% endhighlight %}

而这一workaround，就是给inline hook带来麻烦的根源。

## 我遇到的问题

我是在Android x86模拟器下，通过VirtualHoook的MSHook功能，对方法`__system_property_get`进行hook，从而修改应用所获取到的设备属性。但是，实际运行发现，一旦在hook方法中调用backup方法，就会出现内存异常访问。

于是，通过调试和反编译，我找到了问题的原因。方法`__system_property_get`的起始几条指令正如上面列出的：

{% highlight asm %}
.text:0002ACC7                 push    ebx
.text:0002ACC8                 sub     esp, 18h
.text:0002ACCB                 call    __x86_get_pc_thunk_bx
.text:0002ACD0                 add     ebx, 0CDA4Ch
{% endhighlight %}

然而，在进行inline hook时，由于前两条指令总长度只有4 bytes，达不到绝对跳转所需要的5 bytes，所以origin方法的前3条指令都被复制到了backup中（当然，这里有将`call`替换为`jmp`的操作）。

但是，在调用backup方法时，第3条指令`call __x86_get_pc_thunk_bx`将`ebx`设置为backup所在代码的`pc`；随后，跳转回origin执行第4条指令。细心的读者一定发现了，此时通过`ebx`进行基于`pc`的相对偏移访问会出现错误。根本原因就是在backup方法中设置的`ebx`，并不是origin方法真正希望得到的值。

## 如何解决

如何解决这一问题？

我现在能想到的，就是在将origin的指令复制到backup时，进一步检查其是否存在类似于`__x86_get_pc_thunk_bx`这样的坑；如果存在，那么就需要对`ebx`加上相应的偏移量之后，再跳转回origin。

在网上搜索后，似乎也没有找到有人遇到类似的问题。希望哪位大牛能够指点一下。


__5月19日更新__: 按照上述思路，实现了一个基本的解决方案。具体地，在将指令复制到backup时，检查其是否存在`call`指令？如果存在，其目标方法是否是`__x86_get_pc_thunk_bx`或`__x86_get_pc_thunk_cx`？如果是，那么在后面再加上一条`sub ebx 0xXXXXXXXX`这样的指令，减去origin方法和backup方法之间的偏移量，从而修整相应的寄存器。具体代码可见[这里](https://github.com/rk700/VirtualHook/commit/70ecbe5b878595cefe0b2b742283f1a42ba075f1#diff-6a849b4a951a45abcae8451bb105af62R98)和[这里](https://github.com/rk700/VirtualHook/commit/70ecbe5b878595cefe0b2b742283f1a42ba075f1#diff-6a849b4a951a45abcae8451bb105af62R180)

__5月19日更新2__: 其实ARM架构下做方法inline hook也可能出现这类问题。当目标函数所在的ELF是PIC(Position Independent Code)时，为了访问`.got`，就需要通过`adr`指令进行相对于`pc`的内存访问(pc-relative addressing)。例如：

{% highlight asm %}
.plt:0000FA68 j___system_property_find 
.plt:0000FA68                 ADRL            R12, 0x56A70
.plt:0000FA70                 LDR             PC, [R12,#(__system_property_find_ptr - 0x56A70)]! ; __system_property_find
{% endhighlight %}

不过，一般来说ARM架构下inline hook造成backup方法调用失败的情况要少。因为一般的方法prolog部分已经足够用于跳转到hook方法了，所以复制到backup中的指令，基本上只有这些prolog，故不会带来副作用。