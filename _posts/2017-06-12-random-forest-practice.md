---
title: 机器学习在侵权微信公众号识别中的应用
author: rk700
layout: post
catalog: true
tags:
  - machine learning
---

## 研究背景

最近遇到了收集侵权的微信公众号的需求。根据前期沟通，整理得到了一批搜索关键词以及白名单。

于是，我对之前的公众号信息采集插件进行了调整优化，加快了搜索速度，得到了一批数据：

| 搜索关键词数量 | 118 |
| 采集字段 | 微信ID、公众号名称、账号主体、功能描述、是否通过微信认证、logo地址 |
| 得到的公众号总数 | 7425 |
| 匹配白名单得到的公众号数量 | 534 |

接下来的任务，就是从近7000条公众号中，筛选出仿冒侵权的公众号。

## 基本思路

首先，我们需要对数据进行处理，通过中文分词将原始内容转换为文本向量。随后，使用机器学习算法对数据进行标注。最后，根据标注结果得到疑似侵权公众号。

#### 分词

分词是自然语言处理的重要一步，它可以将原本的一段话分割成词汇。分词后可将文本内容转换为文本向量用于后续的机器学习，因此分词是整个环节中非常重要的一步。

具体地，由于中文不像英文那样有自然的空格分词，中文分词通常需要特殊的工具。常用的python中文分词包有jieba, THULAC等，我所使用的是[jieba分词](https://github.com/fxsjy/jieba)，支持词性标注，支持自定义词典。自定义词典是非常重要的功能，因为除了算法的区别，分词时所使用的词典也会对最终的结果带来很大的影响。

例如，以下是一段示例分词代码：

{% highlight python %}
import jieba
import jieba.posseg as pseg
jieba.load_userdict('/Users/liuruikai756/work/fakeWechat/dict') #加载自定义词典

string='当你不再需要某个实例后，但是这个对象却仍然被引用'
words = pseg.cut(string)
for word, flag in words:
    print('%s %s' % (word, flag))
{% endhighlight %}

输出为：

<pre>
当 p
你 r
不再 d
需要 v
某个 r
实例 n
后 f
， x
但是 c
这个 r
对象 n
却 d
仍然 d
被 p
引用 v
</pre>

可以看到，上面的一段文本被分割成词，并对每个词进行了词性标记，例如`v`代表动词，`n`代表名词，等等。完整的词性标记集可参考[这里](https://gist.github.com/luw2007/6016931)。如果对某些分词的结果不满意，可以使用自定义词典。自定义词典的编写规范也很简单，每行一个词，后面可以加上这个词的词频和词性，[通过调整其词频来增强歧义纠错](https://github.com/fxsjy/jieba/issues/14)。

回到公众号本身，我选取了公众号名称和功能描述这两个字段进行分词，舍弃了单字，并且只保留了名词和动词。由于jieba词典中某些词的词性有误，有些词具有多种词性，而词典只支持一种词性，所以根据词性过滤可能会带来一些偏差，但基本能够分割出想要的结果，一些常见的停用词，如`的` `地` `得`，都可以被舍弃。

#### 构建文本向量

当我们对全部公众号的名称和描述字段进行分词后，就可以将每个公众号的分词结果转换为向量。常用的向量化方式有TF-IDF，其思路是统计每条文本中各个词的词频(TF)，并通过这个词在全部文本中出现次数进行权重调整(IDF)，最终还可对得到的结果进行正则化。这样处理之后，每条文本就转换成了一个稀疏向量，向量中为0的元素代表这个词在这段文本中没有出现，非0元素则代表这个词在文本中的TF-IDF值。

计算TF-IDF，使用的是scikit-learn，这是一个python的机器学习工具包，其中实现了大量常用的机器学习算法。但是scikit的TF-IDF对中文分词支持不好，所以为了使用之前jieba分词的结果，我们将分词结果通过空格再拼接成一段文本，这样TF-IDF在分词时，通过空格分词得到的结果就正好是jieba分词的结果了。

使用scikit的TF-IDF示例代码如下：

{% highlight python %}
from sklearn.feature_extraction.text import TfidfVectorizer

tfidfV = TfidfVectorizer(use_idf=False, norm=None, binary=True, max_df=0.1)
tfidf = tfidfV.fit_transform(corpus)
words = tfidfV.get_feature_names()
{% endhighlight %}

这里的corpus就是语料，是一个`list`，其每个元素是经过前述处理后的每条文本。`TfidfVectorizer`的构造中设定了若干参数，其设定原因接下来会具体介绍。最终得到的`tfidf`是一个稀疏矩阵，其每行对应的便是每条文本的文本向量；`words`则是包含全部词的list，其下标与`tfidf`中的列对应。

从直觉上来讲，在分类时我们关心的是某个词是否出现，而这个词出现了多少次、整个向量是否有进行正则化等因素，则不起决定作用。因此，这里的TF-IDF进行了弱化：一旦某个词在文本中出现，那么这条文本的向量对应于这个词的元素为1，否则为0。所以，我们只需要判断这个词是否出现(`binary=True`)，不计算词频，也不使用IDF对词频进行调整(`use_idf=False`)，最终也不对向量进行正则化(`norm=None`)，而这实际上就等效于one-hot representation。

此外，还设置了`max_df=0.1`，即在文本中出现比例大于10%的词被忽略，这样可以再过滤掉一部分对分类没有意义的词。

#### 训练和调优

现在，数据已经准备完成，接下来可以开始进行机器学习了。我选取的算法是随机森林，随机森林是一组决策树构成的分类器，其学习速度快，易于解释，而且减少了单个决策树带来的过拟合。

首先，需要选取一部分数据作为训练数据。由于公众号总数有7千多，我随机选取了其中的800条，约10%的数据，作为训练数据。对这800条公众号，人工进行了类别标记：将无关的标记为0，相关的（包括真正属于的和仿冒的）标记为1。

使用scikit构造随机森林并训练也是很简单的，示例如下：

{% highlight python %}
from sklearn.ensemble import RandomForestClassifier

clf = RandomForestClassifier(n_estimators=200, max_features=850, n_jobs=-1)
clf.fit(trainingData, trainingLabel)

rfRes = clf.predict(tfidf)
{% endhighlight %}

其中的`trainingData`和`trainingLabel`便是训练数据和其标记，调用`clf.fit()`便完成了训练，由于数据量小，速度还是很快的。训练完成后，就可以调用`clf.predict(tfidf)`对数据进行分类，得到分类结果。

这里，构造的随机森林分类器`RandomForestClassifier`对两个参数进行了调优：`n_estimators`是森林中决策树的数量，默认为10；`max_features`是分类使用的最大特征数量，默认为特征总量的平方根。一般来说，决策树数量越多越好，但太多会增加运算时间，而且数量达到一定程度后准确度变化不大；`max_features`则是影响最终分类效果的一个非常重要的因素。

调优参数时，我使用的是k-fold交叉验证，即将数据分为k份，每次取其中的k-1份作为训练，剩下的1份作为验证，如此进行k轮，得到k个得分，进而得到一个综合的评判结果。交叉验证在scikit中也有现成的实现，示例代码如下：

{% highlight python %}
from sklearn.model_selection import cross_val_score

mf = 800
for i in range(10):
    mf_i = mf+i*20
    clf = RandomForestClassifier(n_estimators=200, max_features=mf_i, n_jobs=-1)
    scores = cross_val_score(clf, trainingData, trainingLabel, cv=8, n_jobs=-1)
    print("max_features %d, Accuracy %0.4f (+/- %0.4f)" % (mf_i, scores.mean(), scores.std()))
{% endhighlight %}

这里我对参数`max_features`进行调优，使用训练数据进行交叉验证，k=8(`cv=8`)，对得到的结果计算其平均值和标准差，作为评判标准。通过试验发现，`max_features`取值为850左右比较合适，此时的准确度较高(0.92)，且标准差比较小。

于是，最终使用的是`n_estimators=200`，`max_features=850`的随机森林，对全部结果进行分类。对于分类为1，即相关的公众号，再通过白名单进一步判断：如果不在白名单中，则为疑似仿冒公众号。由此，得到了约200条疑似仿冒公众号。

## 总结

通过这次对机器学习的试验学习，我有以下体会：

1. scikit可满足大部分基本算法需求，简单易用，在数据量不大的情况下完全胜用
2. 数据和算法一样重要，中文分词的效果对最终分类结果的影响非常大，所以后续优化分类结果可以从优化分词结果入手

此次主要试验了随机森林，后续可以考虑再实验学习其他分类器，如GBDT, SVM或神经网络等。
