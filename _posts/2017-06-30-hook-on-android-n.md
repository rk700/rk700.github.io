---
title: 在Android N上对Java方法做hook遇到的坑
author: rk700
layout: post
catalog: true
tags:
  - android
  - binary
---

之前编写的Android hook工具[YAHFA](http://rk700.github.io/2017/03/30/YAHFA-introduction/)，在Android N之前的环境上运行基本是没有什么问题的。但是，在Android N之后，同样的代码会造成应用崩溃，并且可以稳定复现。为了解决这一问题，我对Android N引入的新机制进行了一定的研究，并针对性地做了修复。由于网上关于Android N混合编译及方法hook的资料不多，这里简要记录下近期学习的内容和修复YAHFA的思路。

---

## 混合编译

相比之前，Android N引入的最大变化就是混合编译，JIT再次回归。于是，在Android N上方法执行的基本流程就如下图所示：

![](https://source.android.com/devices/tech/dalvik/images/jit-workflow.png)

所以，当一个方法被第一次执行时，会进行以下操作：

1. 如果方法的代码已经通过AOT编译，那么就从`.oat`文件加载并运行对应的机器指令
2. 否则，方法将通过interpreter解释执行

针对此变化，`ArtMethod`结构体也进行了相应的调整：之前的字段`entrypoint_from_interpreter_`被移除了，非jni方法的执行通过`entrypoint_from_quick_compiled_code_`进行分发。具体而言，如果方法已经AOT编译完成，那么该entrypoint保存的仍然是实际机器指令的地址；但如果方法尚未被编译，那么就通过`art_quick_to_interpreter_bridge`进入interpreter执行。

如果方法通过interpreter执行，最终会调用到`interpreter::EnterInterpreterFromEntryPoint()`，在这里会调用JIT的方法`NotifyCompiledCodeToInterpreterTransition()`：

![]({{ site.url }}/img/in-posts/hook-on-android-n/NotifyCompiledCodeToInterpreterTransition.png){:.myImage}

继续跟进`NotifyCompiledCodeToInterpreterTransition()`，发现其最终会调用方法`Jit::AddSamples()`。在这里，会取出`ArtMethod`的一个字段`hotness_count_`，并对其进行更新：

![]({{ site.url }}/img/in-posts/hook-on-android-n/AddSamples.png){:.myImage}

而这里，便是Android N混合编译的关键所在。`hotness_count_`这个字段，用于记录方法被调用的"热度"，随着方法调用次数越多，"热度"也越大。除了更新，`AddSamples()`还会检查`hotness_count_`。当其高于`warm_method_threshold_`，方法会变为"warm"的，并记录profile，用于设备空闲时进行AOT编译：

![]({{ site.url }}/img/in-posts/hook-on-android-n/warm_method_threshold_.png){:.myImage}

而如果方法调用次数继续增加，`hotness_count_`高于`hot_method_threshold_`，方法就会进一步变为"hot"的，此时会异步将其加入JIT编译任务中：

![]({{ site.url }}/img/in-posts/hook-on-android-n/hot_method_threshold_.png){:.myImage}

这里提一句题外话，由于非jni方法的`entrypoint_from_JNI_`没有意义，所以在Android N上，这个字段被用于保存上面提到的方法的profile：

![]({{ site.url }}/img/in-posts/hook-on-android-n/profile.png){:.myImage}

回到`hotness_count_`，关于上面提到的热度的threshold，可以参考[官方文档](https://source.android.com/devices/tech/dalvik/jit-compiler)：

![]({{ site.url }}/img/in-posts/hook-on-android-n/threshold.png){:.myImage}

---

## YAHFA针对混合编译的调整

#### 不使用`entrypoint_from_JNI_`

如之前所说，Android N上`ArtMethod`的字段`entrypoint_from_JNI_`被用于保存方法的profile，所以YAHFA之前那种利用`entrypoint_from_JNI_`保存hook信息的方式，就会带来问题。现在，我将hook的方式调整为只修改`entrypoint_from_quick_compiled_code_`，将其指向准备好的一片内存，在那里完成对`eax`的[修正和跳转](https://github.com/rk700/YAHFA/blob/master/library/src/main/jni/HookMain.c#L43)，从而不再对`entrypoint_from_JNI_`进行修改。

#### 清空`hotness_count`

虽然Android N引入了混合编译，但运行时动态加载的dex，还是完全AOT编译后再运行的，这一点可以通过`getprop`确定：

![]({{ site.url }}/img/in-posts/hook-on-android-n/getprop.png){:.myImage}

所以，YAHFA动态加载的hook代码，一定是已经编译为机器指令的。通过native方式替换目标方法，再执行时目标方法时，就会执行hook方法的机器指令，而无需考虑目标方法原本是编译执行还是解释执行。

但是，如果除了hook，还需要执行原方法时，就可能会出现问题。因为YAHFA执行原方法的方式，是将原方法的内容全部备份保存在placeholder中。因此，如果原方法尚未AOT编译，而需要解释执行，那么通过YAHFA执行原方法，就会造成placeholder中的`hotness_count_`增加。

以下是一个简单的实验，通过YAHFA对某方法进行了hook，并在hook方法中执行保存的原方法。在执行hook方法之前，placeholder方法的`ArtMethod`内容如下：

![]({{ site.url }}/img/in-posts/hook-on-android-n/artmethod1.png){:.myImage}

其`hotness_count_`字段偏移为18，长度为2 Bytes，可以看到在没有执行时，其值为0。

当执行完第一次后，`ArtMethod`的内容变化为下：

![]({{ site.url }}/img/in-posts/hook-on-android-n/artmethod2.png){:.myImage}

可以看到`hotness_count_`的值变为了0x181。

当执行完第二次后，`ArtMethod`的内容变化为下：

![]({{ site.url }}/img/in-posts/hook-on-android-n/artmethod3.png){:.myImage}

可以看到`hotness_count_`的值变为了0x302。因此，每执行一次，`hotness_count_`的值就增加0x181。

如果对`hotness_count_`不管，那么多次执行后，会触发对原方法的编译，进而可能出现[IncompatibleClassChangeError](https://github.com/rk700/YAHFA/issues/9)。因此，我选择在每次执行原方法时，都将对应的`hotness_count_`[清空为0](https://github.com/rk700/YAHFA/blob/master/library/src/main/jni/HookMain.c#L60)，这样虽然可能造成性能上的损耗，但是确保了方法不会被编译，进而不会造成更多的坑。

## 垃圾回收造成的问题及修复

除了混合编译，Android N的其他变化也为hook造成了影响。虽然没有具体的文档，但就我的感觉，Android N的垃圾回收似乎更加激进了，其结果就是动态加载的hook代码也可能被垃圾回收掉，而这是在Android N之前没有遇到过的。

以下是使用Android Studio的内存分析工具对heap的分析结果。当demo应用启动并且在前台运行时，可以看到hook方法所在的类是存在的：

![]({{ site.url }}/img/in-posts/hook-on-android-n/fg.png){:.myImage}

然后，按home键将应用切换到后台，很快就观察到应用进行了垃圾回收：

![]({{ site.url }}/img/in-posts/hook-on-android-n/gc.png){:.myImage}

此时再检查heap的情况，发现hook方法所在的类消失了：

![]({{ site.url }}/img/in-posts/hook-on-android-n/refg.png){:.myImage}

于是，此时再执行hook方法，应用就会[崩溃](https://github.com/rk700/YAHFA/issues/8)。

被垃圾回收，说明hook代码没有被引用到。具体而言，YAHFA是通过`DexClassLoader`加载hook插件的dex，再通过反射得到hook方法，最后通过native层对`ArtMethod`的entrypoint进行替换。而这一系列操作，并没有把hook方法和原方法引用起来。这样的结果，就是动态加载的方法，在YAHFA完成hook后，便没有任何引用了。虽然Android N之前没有出现过被垃圾回收的情况，但是在Android N上就被回收了。

所以，解决问题的思路，就是为hook方法添加引用。现在我采取的措施，是在JNI层为传入的hook方法`jobject`调用`NewGlobalRef()`，强制其不被垃圾回收。检查此时的heap，发现在垃圾回收后，hook方法就可以仍然存活了：

![]({{ site.url }}/img/in-posts/hook-on-android-n/gr.png){:.myImage}

由于对Java虚拟机内部机制和垃圾回收不熟悉，这样做可能不够好，希望有熟悉的朋友可以提出更好的方案。

## 总结

Android N带来的变化还是很多的，除了上面提到的，还有更激进的代码优化策略，这些都对hook的稳定性造成影响，特别通过是native方式进行hook，如Xposed, AndFix, Legend，以及YAHFA。所以，YAHFA目前对于Android N的支持可能还有不完善的地方，坑肯定会有很多，需要继续分析研究。

**EDIT:** 发现可以为`ArtMethod`设置`kAccCompileDontBother`这个flag，这样的作用是告诉系统，不要去JIT编译这个方法，从而一劳永逸的解决了上面提到的问题，不需要再通过trampoline中每次调用时清空`hotness_count_`了。
