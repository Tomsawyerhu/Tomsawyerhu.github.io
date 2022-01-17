---
layout: page
title: PA-POINTER-ANALYSIS-3
ftitle: Program Analysis
tags: ["指针分析","非流敏感"]
---

## 指针分析2
本节介绍的是引入method call的指针分析(非流敏感)，也称过程间指针分析。

过程间指针分析和过程内指针分析的区别在于：考虑方法调用对指针流的影响。那具体有哪几方面的影响呢？

+ receive object（this指针）
+ 形参与实参
+ 返回值

指针流规则可以用以下形式化方法表述：

|类型|语句|规则|PFG Edges|
|:----:|-----|------|-----|
|Call|l:r=x.k(a<sub>1</sub>,a<sub>2</sub>,...,a<sub>n</sub>)|O<sub>i</sub>&nbsp;∈&nbsp;pt(x),&nbsp;m=Dispatch(O<sub>i</sub>,k)<br>O<sub>u</sub>&nbsp;∈&nbsp;pt(a<sub>j</sub>),&nbsp;1≤j≤n<br>O<sub>v</sub>&nbsp;∈&nbsp;pt(m<sub>ret</sub>)<br> ————————————<br>O<sub>i</sub>&nbsp;∈&nbsp;pt(m<sub>this</sub>)<br>O<sub>u</sub>&nbsp;∈&nbsp;pt(m<sub>pj</sub>),&nbsp;1≤j≤n<br>O<sub>v</sub>&nbsp;∈&nbsp;pt(r)|a<sub>j</sub>&nbsp;→&nbsp;m<sub>pj</sub>,&nbsp;1≤j≤n<br>m<sub>ret</sub>&nbsp;→&nbsp;r|

可以看见，一共有3组指针流向关系：
+ 调用点对象指针 → receive object 指针
+ 形参指针 → 实参指针
+ 返回值指针 → 返回点左值指针

但是，在PFG Edges中只存在两种边，这是为什么呢？

{% highlight js %}

class A{
  public void f(){}
}

class B extends A{
  public void f(){}
}

class C extends A{
  public void f(){}
}

pt(x) = {new A(), new B(), new C()}
x.f()

{% endhighlight %}

在代码片段中，B与C都继承自A; 当调用x.f时，如果Dispatch到B.f或C.f中还说得通，毕竟子类可以看做基类对象；但是如果Dispatch到A.f中，显然传入的对象类型不可能是B或C, 这种流动是非法的。

那么应该怎么做呢？

其实只需要将x的指针集中新增的对象直接传递给receive object即可。

此外，基于CHA的callgraph算法并不很准确，存在一种基于指针分析的callgraph算法，它比CHA算法更加准确。

这就需要我们将建立callgraph的过程和指针分析过程同步进行。

## 伪代码
![](/public/pic/program_analysis/8.png)
![](/public/pic/program_analysis/9.png)

总体思路：
+ 处理UnReached Method:
  + 标记为Reached
  + 初始化WorkList和PFG
+ 对工作集中的一项：
  + 指针集传播
  + load、store语句：更新PFG
  + call语句：
    + 找到实际调用方法
    + 将this指针的flow操作加入工作集
    + 如果之前没有处理过这个方法，处理UnReached Method

可以看到，对load、store、call语句的处理其实都涉及新指针的产生，这会导致PFG更新。

注意几点：
+ 不同对象，同一类型调用同一方法
+ 同一对象调用同一方法

算法流程最后的产出物包括：
+ 指针与指针集
+ Callgraph
