---
title: 使用Ghidra P-Code对OLLVM控制流平坦化进行反混淆
author: rk700
layout: post
catalog: true
tags:
  - binary
  - reverse
---

**本文同步自[银河实验室博客](http://galaxylab.com.cn/%e4%bd%bf%e7%94%a8ghidra-p-code%e5%af%b9ollvm%e6%8e%a7%e5%88%b6%e6%b5%81%e5%b9%b3%e5%9d%a6%e5%8c%96%e8%bf%9b%e8%a1%8c%e5%8f%8d%e6%b7%b7%e6%b7%86/)。**

## 简介

[OLLVM](https://github.com/obfuscator-llvm/obfuscator)的控制流平坦化是一种常见的代码混淆方式，其基本原理是添加分发器来控制流程的执行。针对这种混淆方式的还原也有了许多研究和工具，大致思路分为动态和静态两种：

- 动态：通过Unicorn模拟执行的方式获取各真实块的关系
- 静态：通过符号执行、中间语言分析等方式获取真实块之间的关系。

静态方式进行OLLVM反混淆的工具，许多是依赖于Binary Ninja, IDA Pro等商业软件提供的中间语言API接口。恰好银河实验室今年来对Ghidra这一开源工具进行了较深入的学习和研究，我们实验发现，通过Ghidra的P-Code这种中间语言，也可以实现控制流平坦化的反混淆。

## 基本思路

控制流平坦化反混淆的技术原理，已经有很多技术文章介绍分享过了，我们参考的是[RPISEC](https://rpis.ec/blog/dissection-llvm-obfuscator-p1/)和[Quarkslab](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)的博客文章。如果对这部分内容比较熟悉，可以直接看下一节的具体实现。

简单来说，OLLVM会添加一个用于控制跳转的状态变量和分发器：当一个真实块执行完成后，会把状态变量的值进行更新，回到分发器进行检查，再根据状态变量的值跳转到下一个真实块执行；如果原本的执行流程中存在条件跳转，则会在各条件下对状态变量设置不同的值，再回到分发器进行检查和跳转。

例如，以下是[Quarkslab文章](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)中的示例C代码：

```c
unsigned int target_function(unsigned int n)
{
  unsigned int mod = n % 4;
  unsigned int result = 0;

  if (mod == 0) result = (n | 0xBAAAD0BF) * (2 ^ n);

  else if (mod == 1) result = (n & 0xBAAAD0BF) * (3 + n);

  else if (mod == 2) result = (n ^ 0xBAAAD0BF) * (4 | n);

  else result = (n + 0xBAAAD0BF) * (5 & n);

  return result;
}
```

使用OLLVM 4.0编译，开启控制流平坦化混淆：

```bash
clang -mllvm -fla ~/tmp/test.c -c -o ~/tmp/test.o
```

将编译得到的test.o文件在Ghidra中加载打开，可以看到左下角的基本块就是分发器（跳转到分发器），其右上侧的各基本块是真实块。更新了状态变量的值后，按照蓝色的箭头线，跳转到左下角进入分发器；而通过分发器的检查后，再根据状态变量的不同的值，按照绿色的箭头线，跳转到下一个真实块。

![]({{ site.url }}/img/in-posts/ghidra-deobf/obf-graph.png){:.myImage}

因此，我们需要对混淆后的代码进行分析，获取以下信息：

- 状态变量的值到真实块的映射，也就是说，如果要跳转到某个真实块，状态变量应该是什么
- 修改更新状态变量的操作，通常都是在真实块的末尾处

如果有了以上信息，那么当状态变量更新后，通过分发器跳转到的下一真实块也就知道了。我们可以将这两个真实块直接连接起来，不需要通过分发器中转，从而实现执行流程的还原。

例如，如果要从函数`target_function()`中返回，需要跳转到`0x1001f0`这个块，其前提条件是在`0x1000c4`中的检查：需要状态变量的值等于`0x57ff42c3`：

![]({{ site.url }}/img/in-posts/ghidra-deobf/obf-to-return.png){:.myImage}

而在`0x1001e4`这个块中，将状态变量的值设置为`0x57ff42c3`，并跳转回分发器。

![]({{ site.url }}/img/in-posts/ghidra-deobf/obf-set-var.png){:.myImage}

由此可知，`0x1001e4`这个块的执行流程会最终到`0x1001f0`即函数返回，因此我们可以跳过中间的分发器，直接将这两个真实块通过跳转连接起来。

## 具体实现

我们使用的是Ghidra的P-Code来对二进制代码进行分析，P-Code的介绍和获取可以参考之前实验室[小朱哥的文章](http://galaxylab.com.cn/%e4%bd%bf%e7%94%a8ghidra-p-code%e8%bf%9b%e8%a1%8c%e8%be%85%e5%8a%a9%e9%80%86%e5%90%91%e5%88%86%e6%9e%90/)。

### 定位状态变量

整个控制流平坦化都是围绕着状态变量来进行的，所以首先需要定位这个状态变量。按照OLLVM的源码，在函数起始处，会对状态变量赋值进行初始化，例如：

![]({{ site.url }}/img/in-posts/ghidra-deobf/state-var-init.png){:.myImage}

我们可以通过人工识别定位到状态变量初始化所在的汇编指令，并通过Ghidra的API获取对应的P-Code：通常初始化状态变量所对应的P-Code是[COPY](https://ghidra.re/courses/languages/html/pcodedescription.html)，将常量复制到目标varnode中。

Ghidra的P-Code遵循Static Single Assignment(SSA)形式，即每个变量只会赋值一次。如果变量可能有不同的值（例如OLLVM混淆中的状态变量），则通过phi node来处理：phi node有多个输入，每个输入对应不同情况下变量的值，输出则是真正的变量。由此，变量就只会在phi node处进行一次赋值，从而满足SSA的要求。在Ghidar P-Code中，通过指令[MULTIEQUAL](https://ghidra.re/courses/languages/html/additionalpcode.html)来对应phi node。

因此，前面提到的初始化状态变量的`COPY`指令，复制的目标varnode其实是一个临时变量，作为phi node的输入。我们定位到`COPY`之后，还需要继续跟踪其输出，找到`MULTIEQUAL`即phi node，这个phi node的输出才是真正的OLLVM状态变量：

```python
    # find the pcode for COPYing const

    while pcode_iterator.hasNext():
        pcode = pcode_iterator.next()
        logging.debug('finding COPY const pcode: %s' % pcode)
        if pcode.getOpcode() == PcodeOp.COPY and pcode.getInput(0).isConstant():
            break

    logging.info('COPY const pcode: %s' % pcode)

    # find the state var in phi node

    depth = 0
    while pcode is not None and pcode.getOpcode() != PcodeOp.MULTIEQUAL:
        logging.debug('finding phi node: %s, depth %d' % (pcode, depth))
        if pcode.getOutput() is None:
            logging.warning('output is None in %s' % pcode)
            break
        pcode = pcode.getOutput().getLoneDescend()
        if depth > 5:
            break
        depth += 1
```

### 获取状态变量到真实块的映射

接下来，我们需要获取状态变量到真实块的映射关系。一般来说，我们可以检查状态变量被读取的地方，判断其是否有与常量的比较。不过对Ghidra试验后发现这样有时会缺失一些信息。

因此，我们换用另一种方式：检查所有的条件跳转，如果跳转条件是状态变量等于某个常量，那么就可以将这个常量与跳转目标关联映射起来。

在Ghidra中，条件跳转是通过`CBRANCH`指令完成的，所以我们检查所有的基本块，判断是否为条件跳转：

```python
    for block in high_function.getBasicBlocks():
        # search for conditional jump

        if block.getOutSize() != 2:
            continue

        last_pcode = get_last_pcode(block)
        if last_pcode.getOpcode() != PcodeOp.CBRANCH:
            continue
```

OLLVM混淆的分发器，会检查状态变量是否等于某些常量。因此我们检查`CBRANCH`的跳转条件是否是`INT_NOTEQUAL`或者`INT_EQUAL`，并且进行比较的两个值其中一个是常量，另一个是状态变量：

```python
        if not condition_type in (PcodeOp.INT_NOTEQUAL, PcodeOp.INT_EQUAL):
            continue

        in0 = conditionPcode.getInput(0)
        in1 = conditionPcode.getInput(1)

        if in0.isConstant():
            constVar = in0
            comparedVar = in1
        elif in1.isConstant():
            constVar = in1
            comparedVar = in0
        else:
            logging.debug('not const var in comparision, skipped')
            continue
```

由此，我们就可以获取状态变量的值到跳转目标真实块之间的映射。

### 获取状态变量的更新

接下来，我们需要获取所有修改更新了状态变量的地方。由于状态变量是通过phi node来赋值的，因此我们可以通过回溯phi node的输入，即可找到这些赋值。

对状态变量赋值，也是通过P-Code的`COPY`指令将某个常量复制。所以我们在回溯phi node的输入时，如果遇到了`COPY`指令并且复制的内容是常量，就可以将其视为对状态变量进行赋值更新：

```python
    for state_var_def in phi_node.getInputs().tolist():
        if state_var_def == state_var:
            continue
        pcode = state_var_def.getDef()
        logging.debug('output %s of pcode %s in block %s defines state var' % (state_var_def, pcode, pcode.getParent()))

        find_const_def_blocks(mem, state_var.getSize(), pcode, 0, state_var_defs, None)
```

这些`COPY`指令的所在位置，一般来说就是真实块的末尾。我们将赋值的常量和所在的基本块信息记录下来，用于接下来的控制流还原。

### 还原控制流

有了以上信息，我们就可以还原控制流了。具体分为两种情况：

- 无条件跳转：这种对应于原始代码中没有条件判断的部分，经OLLVM混淆后，修改状态变量的基本块只有一个输出块
- 条件跳转：这种对应于原始代码中有条件判断的部分，经OLLVM混淆后，修改状态变量的基本块有两个输出块

第一种情况比较简单，我们只需要查找状态变量更新后的值所对应的目标块，并将其作为修改状态变量的基本块的后继即可：

```python
        # unconditional jump

        if def_block.getOutSize() == 1:
            const = const_update(const)
            if const in const_map:
                link = (def_block, const_map[const])
                logging.debug('unconditional jump link: %s' % str(link))
                links.append(link)
            else:
                logging.warning('cannot find const 0x%x in const_map' % const)
```

第二种情况稍微复杂一些：如果基本块A修改了状态变量，并且A有两个输出块A1和A2，那么这两个输出块所对应的状态变量的值也是不同的，我们需要分别找到A1和A2的状态变量所对应的后继块，并将其更新为A的输出。如果A1或者A2也修改了状态变量的值，那么就用修改的值来查找后继块；否则，就沿用A所设置的状态变量的值来查找后继：

```python
            # true out block has state var def

            if true_out in state_var_defs:
                true_out_const = const_update(state_var_defs[true_out])
                if true_out_const not in const_map:
                    logging.warning('true out cannot find map from const 0x%x to block' % true_out_const)
                    continue
                true_out_block = const_map[true_out_const]
                logging.debug('true out to block: %s' % true_out_block)

                if false_out in state_var_defs:
                    false_out_const = const_update(state_var_defs[false_out])
                    if false_out_const not in const_map:
                        logging.warning('false out cannot find map from const 0x%x to block' % false_out_const)
                        continue
                    else:
                        false_out_block = const_map[false_out_const]
                        logging.debug('false out to block: %s' % false_out_block)

                # false out doesn't have const def, then use the def in current block for the false out

                elif const in const_map:
                    false_out_block = const_map[const]
                else:
                    logging.warning('mapping of const %s in block %s not found' % (const, def_block))
                    continue
```

### 汇编代码修复

当控制流还原完成后，我们就可以修改相应的汇编代码，完成最终的反混淆。与上面还原控制流类似，这里的修复同样需要对无条件/条件跳转分别进行处理。

无条件跳转较为简单，我们只需要找到基本块中的最后一条指令，将其替换为绝对跳转即可：

```python
        # unconditional jump

        if len(link) == 2:
            target_addr = link[1].getStart().getOffset()
            asm_string = self.patch_unconditional_jump(patch_addr, target_addr)
            logging.debug('patching unconditional jump at %s to %s' % (patch_addr, asm_string))
            patched = self.asm.assembleLine(patch_addr, asm_string)
            if len(patched) > ins.getLength():
                logging.error('not enough space at %s for patch %s' % (patch_addr, asm_string))
                return None
```

条件跳转稍微复杂一些。例如，在x86架构下，是通过`CMOVXX`这条汇编来对状态变量进行修改。因此，我们可以将其更新为`JXX`来实现跳转：

```python
    def patch_conditional_jump(self, ins, true_addr, false_addr):
        op_str = str(ins.getMnemonicString())

        if op_str.startswith('CMOV'):
            return '%s 0x%x\nJMP 0x%x' % (op_str.replace('CMOV', 'J'), true_addr, false_addr)
        else:
            return None
```

这样的修复方式比较直接，没有将多余指令清空为`NOP`，也没有处理修复长度超出可用长度的情况。

## 反混淆效果

对之前示例的`target_function()`进行OLLVM控制流平坦化混淆后，在Ghidra中反编译的代码如下，我们可以很容易地识别出状态变量：

![]({{ site.url }}/img/in-posts/ghidra-deobf/obf.png){:.myImage}

选中初始化状态变量所对应的汇编代码，再运行反混淆脚本，反编译的代码自动更新如下：

![]({{ site.url }}/img/in-posts/ghidra-deobf/deobf.png){:.myImage}

对比反混淆后的伪代码与源代码，发现逻辑是相同的，反混淆成功。

## 小结

本文中介绍的反混淆Ghidra脚本，已经发布至银河实验室的Github仓库[https://github.com/PAGalaxyLab/ghidra_scripts/blob/master/ollvm_deobf_fla.py](https://github.com/PAGalaxyLab/ghidra_scripts/blob/master/ollvm_deobf_fla.py)。试验下来对于未优化的OLLVM控制流平坦化可较好地完成还原，不过如果在OLLVM混淆时开启了较高等级的优化，则反混淆效果会受一定影响。

