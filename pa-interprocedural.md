---
layout: page
title: PA-INTERPROCEDURAL
ftitle: Program Analysis
---

## Content
* [Inter procedural](#inter-procedural)
* [CallGraph](#callgraph)
  + [How to build callgraph?](#how-to-build-callgraph)
  + [Class Hierarchy Analysis](#class-hierarchy-analysis)
  + [Build global callgraph with CHA](#build-global-callgraph-with-cha)
* [Inter-Procedural Analysis](#cross-procedural-analysis)
  + [Interprocedural CFG](#interprocedural-cfg)
  + [Interprocedural Dataflow Analysis](#interprocedural-dataflow-analysis)

## Inter procedural
在过程内分析中，我们对于函数调用采用safe approximation的策略，这样会造成结果不准确，往往起不到好的优化效果。因此，考虑引入过程间分析。

## CallGraph
调用图描述的是程序方法间调用关系，由调用点、调用边组成。

+ 调用点为函数方法
+ 调用边为有向边，由调用点指向调用点，描述的是调用关系

### How to build callgraph?
这里主要考虑OO语言，以java主要分析对象。

构建方法：
* Class hierarchy analysis(CHA)
* Rapid type analysis(RTA)
* Variable type analysis(VTA)
* Pointer analysis(k-CFA)

由上而下，精度变高、速度变慢。

> *附1：java函数调用类型*
>> 在java7以及之前的版本，共有4种调用类型
>>
>> |调用类型|场景|调用点|
>> |-------|-----|-----|
>> |invokestatic|类静态方法|编译时决定|
>> |invokespecial|示例私有方法、父类示例方法、构造方法|编译时决定|
>> |invokeinterface & invokevirtual|其他|运行时决定|
>
> ***
>
> *附2：函数签名*
>
> Signature = Class Type + Method Name + Descriptor（唯一确定方法）
>
> Descriptor = Return Type + Parameter Type
>
> 写作 **<C: T f(P,Q,R)>**
>
> ***
>
> *附3：动态分派(Method Dispatch)*
>
> ● 以调用点(callsite)的对象类型为入口类
>
> ● 若在当前类中找到method name, method descriptor相同，且不是抽象方法的方法，则找到
>
> ● 否则，去父类中寻找（伪代码如下所示）
> ```
> 输入：声明类型，方法签名
> 输出：调用点
>
> Dispatch(c, sig)
>   FOR method IN c DO
>     s ← method.sig
>     IF sig.name = s.name AND sig.descriptor = s.descriptor THEN
>       RETURN method
>     ENDIF
>   ENDFOR
>   p ← c.parent
>   RETURN Dispatch(p, sig)
>
> ```



### Class Hierarchy Analysis
基于继承关系的调用图构建方法(伪代码如下)

```
输入：声明类型，方法签名
输出：调用点集合

Resolve(c, sig)
  callsite_set ← ()
  subclasses ← get_all_subclasses(c)
  FOR cc IN subclasses DO
    callsite_list += Dispatch(cc, sig)
  ENDFOR
  callsite_list += Dispatch(c, sig)
  RERURN callsite_set
```
事实上，函数Resolve能够根据声明类型找到所有可能调用点。

{% highlight js %}

class A{
  public void f(){}
}

class B extends A{
  public void f(){}
}

class C extends B{
  public void f(){}
}

public static void main(){
  A c = new C();
  c.f();
}

{% endhighlight %}

当调用c.f()时，根据Resolve方法，我们可以得出CHA算法计算结果是(**<C: V f()>**, **<B: V f()>**, **<A: V f()>**).

这是因为Resolve方法会找到声明类本身和它的所有子类，对这些类都调用Dispatch方法；但是事实上，这里c.f()的调用点一定是**<C: V f()>**，这也说明了CHA算法本身是不太精确的。

构建完单个方法的调用图，那么如何构建全局范围的调用图呢？

### Build global callgraph with CHA

大部分调用图的构建都是以Main函数为入口的。这种算法的迭代过程很像是一棵树的广度遍历。

以Main函数为起点，对函数体中所有函数调用计算调用关系；更准确地说，这些计算出的callee函数叫做可达方法。然后对这些可达方法进行新一轮迭代，直到没有新的可达方法。

伪代码如下：

```
输入：
输出：
参数：
* WL: WorkList, 工作集；用于存放当前正在分析的方法
* RM: ReachingMethod, 可达方法集合
* CE: CallEdges, 调用边集合；用于记录调用关系

BuildGlobalCallgraph()
  WL ← (main)
  RM ← ()
  CE ← ()
  WHILE WL IS NOT EMPTY DO
    m ← GetOne(WL)
    FOR mm IN Resolve(method_class, method_signature)
      AddOne(CE, m → mm)
      IF mm NOT IN RM
        AddOne(WL, mm)
      ENDIF
    ENDFOR
    AddOne(RM, m)
  ENDWHILE
  RETURN CE
```

## Cross-Procedural Analysis
跨过程分析，也称过程间分析

### Interprocedural CFG
ICFG, 过程间数据流图，由于也是数据流图，因而与过程内数据流图非常相似。
> ICFG = CFGs + call & return edges

过程间数据流图由过程内数据流图、调用边、返回边组成。
首先补充一下概念：
* Callsite: 方法调用点
* Returnsite: callsite的下一句

什么是调用边？

顾名思义，就是由callsite指向目标函数CFG起始节点的有向边。

什么是返回边？

就是由目标函数CFG返回节点指向returnsite的有向边。

如图所示：

![](/public/pic/program_analysis/3.png)

这里函数调用关系来自[CHA算法建立的global callgraph](#build-global-callgraph-with-cha)

### Interprocedural Dataflow Analysis
建立ICFG后, 如何做过程间数据流分析呢？

可以注意到，在上图所示的ICFG, callsite到returnsite的边并没有被抹除。这里涉及到一个新概念——本地数据流（local dataflow）. 本地数据流指的是不包含在被调用函数的入参列表中的数据流，这部分数据不会被被调用函数使用，因而不会改变，可以直接流向返回之后的节点。

与此相对的，是外部数据流，即包含在入参列表中的数据流。这部分数据流会流向被调用函数的起始节点，并从被调用函数的返回节点流出。

在过程间数据流分析中，有如下几种Transfer函数：
* Transfer function = Node transfer + Edge transfer
* Edge Transfer = Call Edge Transfer + Return Edge Transfer
![](/public/pic/program_analysis/4.png)

在Call Edge Transfer和Return Edge Transfer中，需要做数据转换，也就是将入参列表中参数与流入的数据流中数据对应，将返回值与returnsite处的变量对应。

另外，在Call Edge Transfer需要额外做一件事，就是kill left hand variable，也就是图中的y.
![](/public/pic/program_analysis/5.png)
因为y的值由被调用函数返回值决定，因而不能被包括在本地数据流中。
