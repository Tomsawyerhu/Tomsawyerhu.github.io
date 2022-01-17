---
layout: page
title: PA-POINTER-ANALYSIS-2
ftitle: Program Analysis
tags: ["指针分析"]
---

## 指针分析2
本节介绍的是不引入method call的指针分析。

指针分析包括两部分：
+ PFG的建立
+ 对象在PFG中的传播

一般思维中，这两部分是依序进行的；即先建立指针流图、再进行对象传播。
事实上，不仅对象传播依赖于指针流图，指针流图的建立也依赖于对象传播。
![](/public/pic/program_analysis/6.png)
为什么呢？

以[代码片段](#target-code-segment)为例，需要传播的对象有两个(new 语句)，而这些对象中会派生出新的指针(如c.f、d.f)，由load、store语句派生出的指针无法在指针传播之前确定。

因此，这两个阶段是交叉进行的。

大体思路如下：一旦有新的对象加入指针集，那么不仅需要进行指针传播，也需要检查新对象上是否派生出新的指针。

## 伪代码
```
Solve(S)
  WL = [], PFG = {}
  FORREACH i: x = new T() ∈ S DO
    add <x, {oi}> to WL

  FOREACH x = y ∈ S DO
    AddEdge(y, x)

  WHILE WL is not empty DO
    remove <n, pts> from WL
    Δ = pts - pt(n)
    Propagate(n, Δ)
    IF n represents a variable x THEN
      FOREACH oi ∈ Δ DO
        FOREACH x.f = y ∈ S DO
          AddEdge(y, oi.f)
        FOREACH y = x.f ∈ S DO
          AddEdge(oi.f, y)

AddEdge(s, f)
  IF s → t ∉ PFG THEN
    add s → t to PFG
    IF pt(s) is not empty THEN
      add <t,pt(s)> to WL

Propagate(n, pts)
  IF pts is not empty THEN
    pt(n) ∪= pts
    FOREACH n → s ∈ PFG DO
      add <s, pts> to WL
```
<div style="background-color:#F0F8FF;padding:10px">
  <p>
    <b>S &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;</b>
    <i>set of statements of the input program</i>
  </p>
  <p>
    <b>WL &nbsp; &nbsp; &nbsp; &nbsp;</b>
    <i>work list of the changing pointer set</i>
  </p>
  <p>
    <b>PFG &nbsp; &nbsp; &nbsp; &nbsp;</b>
    <i>pointer flow graph representing pointer flow relationship</i>
  </p>
</div>

## Sandbox
### Target Code Segment
<p style="font-size:15px;background-color:#FFE4C4;padding:10px;">b = new C();<br>a = b;<br>c = new C();<br>c.f = a;<br>d = c;<br>c.f = d;<br>e = d.f;</p>

### Display
<div style="background-color:#F0FFF1;">
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step1</b>
    <br><br>
    WL = [ < b: { o1 } > , < c: { o2 } > ]
    <br>
    CFG = { b → a , c → d }
    <br><br>
    <b><i>Annotation: 根据New语句初始化工作集，根据Assign类型语句初始化PFG</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step2</b>
    <br><br>
    WL = [ < c: { o2 } > ]
    <br>
    CFG = { b → a , c → d }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step3</b>
    <br><br>
    WL = [ < c: { o2 } > ]
    <br>
    CFG = { b → a , c → d }
    <br>
    PT(b) = { o1 }
    <br><br>
    <b><i>Annotation: 将Step2中取出项中的对象加入b的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step4</b>
    <br><br>
    WL = [ < c: { o2 } > , < a: { o1 } > ]
    <br>
    CFG = { b → a , c → d }
    <br>
    PT(b) = { o1 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < a: { o1 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step5</b>
    <br><br>
    WL = [ < a: { o1 } > ]
    <br>
    CFG = { b → a , c → d }
    <br>
    PT(b) = { o1 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step6</b>
    <br><br>
    WL = [ < a: { o1 } > ]
    <br>
    CFG = { b → a , c → d }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 }
    <br><br>
    <b><i>Annotation: 将Step5中取出项中的对象加入c的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step7</b>
    <br><br>
    WL = [ < a: { o1 } > , < d: { o2 } > ]
    <br>
    CFG = { b → a , c → d }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < d: { o2 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step8</b>
    <br><br>
    WL = [ < a: { o1 } > , < d: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 }
    <br><br>
    <b><i>Annotation: 由于c.f = a, c.f = d; 需要向CFG中加入新的指针节点</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step9</b>
    <br><br>
    WL = [ < d: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step10</b>
    <br><br>
    WL = [ < d: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 }
    <br><br>
    <b><i>Annotation: 将Step9中取出项中的对象加入a的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step11</b>
    <br><br>
    WL = [ < d: { o2 } > , < o2.f: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < o2.f: { o1 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step12</b>
    <br><br>
    WL = [ < o2.f: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step13</b>
    <br><br>
    WL = [ < o2.f: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 }
    <br><br>
    <b><i>Annotation: 将Step12中取出项中的对象加入d的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step14</b>
    <br><br>
    WL = [ < o2.f: { o1 } > , < o2.f: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < o2.f: { o2 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step15</b>
    <br><br>
    WL = [ < o2.f: { o1 } > , < o2.f: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 }
    <br><br>
    <b><i>Annotation: 由于e = d.f; 需要向CFG中加入新的指针节点</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step16</b>
    <br><br>
    WL = [ < o2.f: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step17</b>
    <br><br>
    WL = [ < o2.f: { o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 }
    <br><br>
    <b><i>Annotation: 将Step12中取出项中的对象加入o2.f的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step18</b>
    <br><br>
    WL = [ < o2.f: { o2 } > , < e: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < e: { o1 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step19</b>
    <br><br>
    WL = [ < e: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step20</b>
    <br><br>
    WL = [ < e: { o1 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 , o2 }
    <br><br>
    <b><i>Annotation: 将Step19中取出项中的对象加入o2.f的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step21</b>
    <br><br>
    WL = [ < e: { o1, o2 } > ]
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 , o2 }
    <br><br>
    <b><i>Annotation: 指针传播，将 < e: { o2 } > 加入工作集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step22</b>
    <br><br>
    WL = []
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 , o2 }
    <br><br>
    <b><i>Annotation: 将工作集中一项取出</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step23</b>
    <br><br>
    WL = []
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 , o2 } , PT(e) = { o1 , o2 }
    <br><br>
    <b><i>Annotation: 将Step22中取出项中的对象加入e的指针集</i></b>
  </p>
  <p style="background-color:#F0FFFF;padding:10px;">
    <b>Step24</b>
    <br><br>
    WL = []
    <br>
    CFG = { b → a , c → d , a → o2.f , d → o2.f , o2.f → e }
    <br>
    PT(b) = { o1 } , PT(c) = { o2 } , PT(a) = { o1 } , PT(b) = { o2 } , PT(o2.f) = { o1 , o2 } , PT(e) = { o1 , o2 }
    <br><br>
    <b><i>Annotation: 工作集为空，过程结束</i></b>
  </p>
</div>

![](/public/pic/program_analysis/7.png)

