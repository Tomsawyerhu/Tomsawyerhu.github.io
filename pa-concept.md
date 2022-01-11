---
layout: page
title: PA-CONCEPT
ftitle: Program Analysis
---

## 静态分析基本概念

### 概念列表
- [CFG](#cfg)
- [May Analysis](#may-analysis)
- [Must Analysis](#must-analysis)
- [INPUT & OUTPUT STATE](#input--output-state)
- [Transfer Function](#transfer-function)
- [SPLIT MEET](#split-meet)
- [Forward Analysis](#forward-analysis)
- [Backward Analysis](#backward-analysis)
- [Basic Block](#basic-block)

### CFG
*CFG*(Control Flow Graph)，控制流图。一个过程或程序的抽象表现，是用在编译器中的一个抽象数据结构，由编译器在内部维护，代表了一个程序执行过程中会遍历到的所有路径。它用图的形式表示一个过程内所有基本块执行的可能流向, 也能反映一个过程的实时执行过程。

![](/public/pic/program_analysis/1.png)

### May Analysis
可能性分析，分析的结果表示为可能性。

例如：变量a在节点p可能被使用，代表变量a的存活性

May Analysis是一种Over-approximation，它输出可能的结果。


### Must Analysis
必然性分析，分析的结果表示为必然性。

例如：变量a在节点p必然是常量，代表变量a的值固定

Must Analysis是一种Under-approximation，它输出必然的结果。

### INPUT & OUTPUT STATE
输入输出状态，静态分析关注的是程序在执行节点之前、之后的状态，而不是程序在节点内部的状态。

### Transfer Function
转换函数，输入在经过转换函数后变为输出，表示为:

{% highlight js %}

OUT[s]=f(IN[s])

{% endhighlight %}

其中，f代表转换函数。

### SPLIT MEET

{% highlight js %}

一般 IN[s2]=OUT[s1]

分叉(SPLIT) IN[s3]=IN[s2]=OUT[s1]

汇聚(MEET) 符号表示^

{% endhighlight %}

![](/public/pic/program_analysis/2.png)



### Forward Analysis
前项分析，以程序执行的顺序为分析顺序。

{% highlight js %}

OUT[s]=f(IN[s])

{% endhighlight %}

### Backward Analysis
反向分析，以程序执行的逆序为分析顺序。

{% highlight js %}

IN[s]=f(OUT[s])

{% endhighlight %}

### Basic Block
基本块，一个基本块内只含有顺序结构，不包含分支、循环结构。


### 指针分析、跨函数分析
* 指针分析：如变量存在别名的情况下，多个变量名指向同一块内存区域
* 跨函数分析：分析域包含多个函数
