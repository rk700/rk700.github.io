---
title: 内联汇编的限制符
author: rk700
layout: post
catalog: true
tags:
  - binary
---

**本文同步自[银河实验室博客](http://galaxylab.com.cn/%e5%86%85%e8%81%94%e6%b1%87%e7%bc%96%e7%9a%84%e9%99%90%e5%88%b6%e7%ac%a6/)。**

## 问题背景

最近，在编写代码时，因为限制用到了内联汇编（inline assembly）。之前对这种在C代码里嵌入汇编的方式了解的并不多，只知道可以通过`asm()`来实现。但是，编写的代码经过编译器`-O2`优化后的代码却出现了问题。

简单的示例代码如下：

```c
#include <stdio.h>

#include <stdlib.h>

int add(int i, int j) {
    asm (
            "mov %rdi, %rax;"
            "add %rsi, %rax;"
       );
}

int main(int argc, char *argv[]) {
    int i = atoi(argv[1]);
    int j = atoi(argv[2]);

    int res = add(i, j);
    printf("%s + %s is %d\n", argv[1], argv[2], res);

    return res;
}
```

这里是编写了一个函数`add()`，里面通过内联汇编，把参数相加。我还按照之前的经验，以为函数的参数会通过`rdi`和`rsi`传入，`rax`的值作为函数的返回结果被使用。

在没有开启编译优化的情况下，上述代码是可以正常运行的：

```
root@debian:/tmp# gcc test.c -o test 
root@debian:/tmp# ./test 1 2
1 + 2 is 3
```

但是，当开启了编译优化后，运行的结果就出错了：

```
root@debian:/tmp# gcc test.c -o test -O2
root@debian:/tmp# ./test 1 2
1 + 2 is 0
```

此时，查看编译得到的二进制代码，发现确实存在问题：

```
0000000000001060 <main>:
    1060:       41 54                   push   %r12
    1062:       ba 0a 00 00 00          mov    $0xa,%edx
    1067:       53                      push   %rbx
    1068:       48 89 f3                mov    %rsi,%rbx
    106b:       48 83 ec 08             sub    $0x8,%rsp
    106f:       48 8b 7e 08             mov    0x8(%rsi),%rdi
    1073:       31 f6                   xor    %esi,%esi
    1075:       e8 c6 ff ff ff          callq  1040 <strtol@plt>
    107a:       48 8b 7b 10             mov    0x10(%rbx),%rdi
    107e:       ba 0a 00 00 00          mov    $0xa,%edx
    1083:       31 f6                   xor    %esi,%esi
    1085:       e8 b6 ff ff ff          callq  1040 <strtol@plt>
    108a:       48 89 f8                mov    %rdi,%rax               # add()
    108d:       48 01 f0                add    %rsi,%rax
    1090:       45 31 e4                xor    %r12d,%r12d
    1093:       48 8b 53 10             mov    0x10(%rbx),%rdx
    1097:       48 8b 73 08             mov    0x8(%rbx),%rsi
    109b:       31 c0                   xor    %eax,%eax
    109d:       44 89 e1                mov    %r12d,%ecx
    10a0:       48 8d 3d 5d 0f 00 00    lea    0xf5d(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    10a7:       e8 84 ff ff ff          callq  1030 <printf@plt>
    10ac:       48 83 c4 08             add    $0x8,%rsp
    10b0:       44 89 e0                mov    %r12d,%eax
    10b3:       5b                      pop    %rbx
    10b4:       41 5c                   pop    %r12
    10b6:       c3                      retq
    10b7:       66 0f 1f 84 00 00 00    nopw   0x0(%rax,%rax,1)
    10be:       00 00
```

可以看到，`108a`处就是被内联优化的方法`add()`，包含了内联汇编的两条指令。但是其参数`rdi`和`rsi`，原本应该是两次调用`strtol()`的返回结果，却并没有看到相关设置；而且`add()`的返回结果`rax`，也没有被最后的`printf()`所使用。因此，造成了计算结果的错误。

## 问题原因

在网上搜索了相关内容，大致理解了编译器这样优化出错的原因。

一般来说，编译器对`asm()`中的汇编代码具体内容，并不会去关注。因为这些汇编代码是用户所编写，编译器会直接保留。因此，像开始的那种写法：

```c
int add(int i, int j) {
    asm (
            "mov %rdi, %rax;"
            "add %rsi, %rax;"
       );
}
```

对于编译器而言，它看到的是一个接受了2个参数的函数`add()`，但这个函数并没有对传入的参数做任何使用，也不知道函数的返回结果是在哪里。这种没有使用传参的函数，按理来说，对传入的任何参数值，其运行的结果应该都是相同的。因此，当编译器在做优化时，就不会再使用额外的指令，把`rdi`和`rsi`设置为所需的参数值。返回结果也是同样的原因，被编译器优化掉了。

因此，我们需要通过某种方式告诉编译器，这个函数所接收的参数在`asm()`的内联汇编中是有读取使用的，而且函数的返回值也在内联汇编中设置完成。这样，即使开启了编译优化选项，也可以得到正确运行的结果。而这种告诉编译器哪些变量、寄存器被使用、返回的方式，就是设置`asm()`中的内联汇编限制符（inline assembly constraints）。


## 内联汇编的限制符

关于限制符的使用，可以阅读[这篇文章](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html#s5)详细了解，这里就只做简单的介绍。

完整的内联汇编格式是 `asm(汇编代码 : 输出表 : 输入表 : 破坏的内容)`。汇编代码后面的内容就是限制符，它告诉了寄存器，前面的汇编代码中，哪些是输出，那些是输入，哪些是运行时被修改破坏(clobbered)的内容。

对于输出表和输入表来说，可以包含多项，用逗号隔开。每项使用`"x"(y)`的形式，其中双引号中的字符串`x`是这一项的类型，括号中的`y`是这一项的符号名称。常见的类型字符串有：

| r | 任意可用的通用寄存器 |
| m | 内存 |
| g | 寄存器、内存或立即数 |
| a | rax |
| b | rbx |
| c | rcx |
| d | rdx |
| S | rsi |
| D | rdi |

例如，`"b"(var)`限制变量`var`应该保存在寄存器`rbx`中。对于输出来说，通常还需要在类型字符串前面加上`=`或者`+`，前者说明是只写的，后者说明是读写的。

对于破坏的内容，则可以直接通过`"%rax"`, `"%rbx"`这种形式告诉编译器哪些寄存器的值被修改了，如果要使用这些寄存器，需要备份其值。除了寄存器，还可以通过`"cc"`说明条件码被修改了（通常是内联汇编代码中有比较大小的指令），通过`"memory"`说明内存被修改了。所有这些项，同样被逗号分隔。

如果设置了输出表或者输入表的内容，在内联汇编代码中，还可以通过`%0`, `%1`这种方式来获取对应的输出、输入项。对应规则是：将输出表中的各项依次排列，再跟上输入表中的各项目，按照从0开始依次编号。例如：

```c
int i, j;
asm(
    "mov %1, %0;"
    : "=r"(j)
    : "r"(i)
    :
    );
```

上述代码中，限制符指定了输出是寄存器中的变量`j`，输入是寄存器中的变量`i`。按照规则，`%0`就是输出的第一项，`%1`就是接下来的输入的第一项。而相应的汇编代码所操作的内容，就是把输入寄存器的值，`mov`到了输出寄存器中。而为了和这种记录方式区别开来，汇编代码中正常的寄存器引用，需要用连续两个`%`符号，例如`%%rax`。


回到最开始的`add()`函数，我们可以为其中的内联汇编添加限制符如下：

```c
    int res;
    asm (
            "mov %%rdi, %%rax;"
            "add %%rsi, %%rax;"
            : "=a"(res)
            : "D"(i), "S"(j)
            :
       );
```

上面这段汇编代码，是从输入的寄存器`rdi`中读取变量`i`、寄存器`rsi`中读取变量`j`，运算后将结果输出到寄存器`rax`中的变量`res`。除了`rdi`, `rsi`, `rax`这三个寄存器，其他寄存器并没有被修改，所以限制符最后破坏的内容为空。

经过这样的修改，我们就可以告诉编译器：`add()`函数的参数是有被内联汇编使用的，不能被优化掉；内联汇编代码的结果会保存在变量`res`中，返回这个变量即可。完整的代码如下：

```c
#include <stdio.h>

#include <stdlib.h>

int add(int i, int j) {
    int res;
    asm (   
            "mov %%rdi, %%rax;"
            "add %%rsi, %%rax;"
            : "=a"(res)
            : "D"(i), "S"(j)
            :
       );
    return res;
}

int main(int argc, char *argv[]) {
    int i = atoi(argv[1]);
    int j = atoi(argv[2]);

    int res = add(i, j);
    printf("%s + %s is %d\n", argv[1], argv[2], res);

    return res;
}
```

此时，开启`-O2`优化选项，得到的运行结果也是正确的：

```
root@debian:/tmp# gcc test.c -o test -O2
root@debian:/tmp# ./test 1 2
1 + 2 is 3
```

查看二进制代码，可以确认此时的指令是正确的：

```
0000000000001060 <main>:
    1060:       41 54                   push   %r12
    1062:       ba 0a 00 00 00          mov    $0xa,%edx
    1067:       53                      push   %rbx
    1068:       48 89 f3                mov    %rsi,%rbx
    106b:       48 83 ec 08             sub    $0x8,%rsp
    106f:       48 8b 7e 08             mov    0x8(%rsi),%rdi
    1073:       31 f6                   xor    %esi,%esi
    1075:       e8 c6 ff ff ff          callq  1040 <strtol@plt>
    107a:       48 8b 7b 10             mov    0x10(%rbx),%rdi
    107e:       ba 0a 00 00 00          mov    $0xa,%edx
    1083:       31 f6                   xor    %esi,%esi
    1085:       49 89 c4                mov    %rax,%r12
    1088:       e8 b3 ff ff ff          callq  1040 <strtol@plt>
    108d:       44 89 e7                mov    %r12d,%edi
    1090:       48 8b 53 10             mov    0x10(%rbx),%rdx
    1094:       89 c6                   mov    %eax,%esi
    1096:       48 89 f8                mov    %rdi,%rax
    1099:       48 01 f0                add    %rsi,%rax
    109c:       48 8b 73 08             mov    0x8(%rbx),%rsi
    10a0:       41 89 c4                mov    %eax,%r12d
    10a3:       89 c1                   mov    %eax,%ecx
    10a5:       48 8d 3d 58 0f 00 00    lea    0xf58(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    10ac:       31 c0                   xor    %eax,%eax
    10ae:       e8 7d ff ff ff          callq  1030 <printf@plt>
    10b3:       48 83 c4 08             add    $0x8,%rsp
    10b7:       44 89 e0                mov    %r12d,%eax
    10ba:       5b                      pop    %rbx
    10bb:       41 5c                   pop    %r12
    10bd:       c3                      retq
    10be:       66 90                   xchg   %ax,%ax
```

可以看到，`1085`处将第一次`strtol()`的结果`rax`保存到`r12`中，在`108d`处将`r12`的值设置给`edi`，作为`add()`的第一个参数。类似地，第二次`strtol()`的结果在`1094`处被设置给`esi`，作为`add()`的第二个参数。这两个参数的设置都没有问题。而`add()`的返回结果，也在`10a3`处设置到`ecx`，正确地作为`printf()`的第4个参数。

## 另一个例子

接下来，我们再看一个例子。开始之前，我们先介绍下纯函数(pure function)和对其的优化。

如果一个函数的运算结果只依赖于传入的参数，而且除了运算结果没有其他的副作用，那么这样的函数就可以称为“纯函数”。如果熟悉Haskell这类函数式编程，那么应该对纯函数不陌生。上文中的方法`add()`就是一个纯函数，用户也可以给函数显式地添加`__attribute__ ((pure))`，告诉编译器某个函数是纯函数。

当编译器遇到一个纯函数，而且这个纯函数有被连续多次以同样的参数调用，那么根据纯函数的性质，编译器认为这多次调用的返回结果都是相同的，而且没有副作用。此时，往往会把多次调用优化为一次。例如：

```c
#include <stdio.h>

#include <stdlib.h>


int add(int i, int j) {
    int res;
    asm (
            "mov %%rdi, %%rax;"
            "add %%rsi, %%rax;"
            : "=a"(res)
            : "D"(i), "S"(j)
            : 
       );
    return res;
}

int main(int argc, char *argv[]) {
    int i = atoi(argv[1]);
    int j = atoi(argv[2]);

    while(1) {
        int res = add(i, j);
        printf("%s + %s is %d\n", argv[1], argv[2], res);
    }

    return 0;
}
```

使用`-O2`优化选项，编译得到的结果是：

```
0000000000001060 <main>:
    1060:       55                      push   %rbp
    1061:       ba 0a 00 00 00          mov    $0xa,%edx
    1066:       53                      push   %rbx
    1067:       48 89 f3                mov    %rsi,%rbx
    106a:       48 83 ec 08             sub    $0x8,%rsp
    106e:       48 8b 7e 08             mov    0x8(%rsi),%rdi
    1072:       31 f6                   xor    %esi,%esi
    1074:       e8 c7 ff ff ff          callq  1040 <strtol@plt>
    1079:       48 8b 7b 10             mov    0x10(%rbx),%rdi
    107d:       31 f6                   xor    %esi,%esi
    107f:       ba 0a 00 00 00          mov    $0xa,%edx
    1084:       89 c5                   mov    %eax,%ebp
    1086:       e8 b5 ff ff ff          callq  1040 <strtol@plt>
    108b:       89 ef                   mov    %ebp,%edi
    108d:       89 c6                   mov    %eax,%esi
    108f:       48 89 f8                mov    %rdi,%rax
    1092:       48 01 f0                add    %rsi,%rax
    1095:       89 c5                   mov    %eax,%ebp
    1097:       66 0f 1f 84 00 00 00    nopw   0x0(%rax,%rax,1)
    109e:       00 00
    10a0:       48 8b 53 10             mov    0x10(%rbx),%rdx
    10a4:       48 8b 73 08             mov    0x8(%rbx),%rsi
    10a8:       89 e9                   mov    %ebp,%ecx
    10aa:       31 c0                   xor    %eax,%eax
    10ac:       48 8d 3d 51 0f 00 00    lea    0xf51(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    10b3:       e8 78 ff ff ff          callq  1030 <printf@plt>
    10b8:       eb e6                   jmp    10a0 <main+0x40>
    10ba:       66 0f 1f 44 00 00       nopw   0x0(%rax,%rax,1)
```

可以看到，在`10b8`处是一个无条件跳转，回到`10a0`处，构成了`while`循环。而纯函数`add()`只在`108f`处有调用过一次，并没有在`while`循环的内部每次调用。这正是编译器把`add()`作为纯函数来优化的结果。

但是，如果`add()`函数中的内联汇编代码并没有构成纯函数，此时就需要小心设置限制符了。例如：

```c
#include <stdio.h>

#include <stdlib.h>


int add(int *i, int j) {
    int res;
    asm (
            "mov 0(%%rdi), %%edx;"
            "mov %%edx, %%eax;"
            "inc %%edx;"
            "mov %%edx, 0(%%rdi);"
            "add %%esi, %%eax;"
            : "=a"(res)
            : "D"(i), "S"(j)
            : "%rdx"
       );
    return res;
}

int main(int argc, char *argv[]) {
    int i = atoi(argv[1]);
    int j = atoi(argv[2]);

    while(i < 5) {
        int res = add(&i, j);
        printf("%s + %s is %d\n", argv[1], argv[2], res);
    }

    return 0;
}
```

此时的`add()`就不是纯函数了，因为其除了从`int *i`中读取并和`j`相加，还会把`int *i`的内存内容加一，这就是它的副作用。按理来说，由于这个副作用的存在，`main()`函数中的`while`循环不会一直运行下去，但优化编译后的结果却并不是这样：

```
0000000000001060 <main>:
    1060:       55                      push   %rbp
    1061:       ba 0a 00 00 00          mov    $0xa,%edx
    1066:       53                      push   %rbx
    1067:       48 89 f3                mov    %rsi,%rbx
    106a:       48 83 ec 18             sub    $0x18,%rsp
    106e:       48 8b 7e 08             mov    0x8(%rsi),%rdi
    1072:       31 f6                   xor    %esi,%esi
    1074:       e8 c7 ff ff ff          callq  1040 <strtol@plt>
    1079:       48 8b 7b 10             mov    0x10(%rbx),%rdi
    107d:       31 f6                   xor    %esi,%esi
    107f:       ba 0a 00 00 00          mov    $0xa,%edx
    1084:       89 44 24 0c             mov    %eax,0xc(%rsp)
    1088:       e8 b3 ff ff ff          callq  1040 <strtol@plt>
    108d:       83 7c 24 0c 04          cmpl   $0x4,0xc(%rsp)
    1092:       7f 3b                   jg     10cf <main+0x6f>
    1094:       48 8d 7c 24 0c          lea    0xc(%rsp),%rdi
    1099:       89 c6                   mov    %eax,%esi
    109b:       8b 17                   mov    (%rdi),%edx
    109d:       89 d0                   mov    %edx,%eax
    109f:       ff c2                   inc    %edx
    10a1:       89 17                   mov    %edx,(%rdi)
    10a3:       01 f0                   add    %esi,%eax
    10a5:       89 c5                   mov    %eax,%ebp
    10a7:       66 0f 1f 84 00 00 00    nopw   0x0(%rax,%rax,1)
    10ae:       00 00 
    10b0:       48 8b 53 10             mov    0x10(%rbx),%rdx
    10b4:       48 8b 73 08             mov    0x8(%rbx),%rsi
    10b8:       31 c0                   xor    %eax,%eax
    10ba:       89 e9                   mov    %ebp,%ecx
    10bc:       48 8d 3d 41 0f 00 00    lea    0xf41(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    10c3:       e8 68 ff ff ff          callq  1030 <printf@plt>
    10c8:       83 7c 24 0c 04          cmpl   $0x4,0xc(%rsp)
    10cd:       7e e1                   jle    10b0 <main+0x50>
    10cf:       48 83 c4 18             add    $0x18,%rsp
    10d3:       31 c0                   xor    %eax,%eax
    10d5:       5b                      pop    %rbx
    10d6:       5d                      pop    %rbp
    10d7:       c3                      retq   
    10d8:       0f 1f 84 00 00 00 00    nopl   0x0(%rax,%rax,1)
    10df:       00 
```

可以看到，在`10cd`处的跳转回`10b0`，对应的就是`while`循环。但是`109b`处被内联优化的`add()`函数仍然在`while`循环外，只执行了一次。这是因为编译器错误地认为`add()`仍然是一个纯函数，而且每次执行`add()`的参数不变，所以优化后的结果就出错了。

我们回到`add()`中的内联汇编限制符来分析原因。`"D"(i)`说明`i`是保存在寄存器`rdi`中，但这里的`i`实际上是一个`int`指针，指向了一片内存区域，所以用`"D"`这种类型字符串，实际上是把`i`是指针这一信息丢失了的。为了解决这一问题，我们可以修改限制符，把`i`的类型用`"m"`（内存）来表示。修改后的完整代码如下：

```c
#include <stdio.h>

#include <stdlib.h>


int add(int *i, int j) {
    int res;
    asm (
            "mov %1, %%rdi;"
            "mov 0(%%rdi), %%edx;"
            "mov %%edx, %%eax;"
            "inc %%edx;"
            "mov %%edx, 0(%%rdi);"
            "add %%esi, %%eax;"
            : "=a"(res)
            : "m"(i), "S"(j)
            : "%rdx", "%rdi"
       );
    return res;
}

int main(int argc, char *argv[]) {
    int i = atoi(argv[1]);
    int j = atoi(argv[2]);

    while(i < 5) {
        int res = add(&i, j);
        printf("%s + %s is %d\n", argv[1], argv[2], res);
    }

    return 0;
}
```

可以看到，现在的`i`是在内存中，而且汇编代码中使用其对应的`%1`指代。此时，编译器知道了有内存地址被传入了内联汇编代码，可能产生副作用，就不会把`add()`方法再视作纯函数优化了：

```
root@debian:/tmp# gcc test.c -o test -O2
root@debian:/tmp# ./test 1 2
1 + 2 is 3
1 + 2 is 4
1 + 2 is 5
1 + 2 is 6
```
