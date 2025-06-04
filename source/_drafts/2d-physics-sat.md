---
title: 2D 碰撞檢測演算法 - SAT(Separating Axis Theorem)
categories: 2D Physics
tags:
---

2D 碰撞檢測中，圓形和 AABB 可以很快的計算出碰撞結果，但在許多場合需要處理的是 OBB 或是多邊形，這時候就需要其他演算法來幫助我們，SAT(Separating Axis Theorem) 是其中一種，中文可稱作分離軸定理。

<!-- more -->
### 1.原理
SAT 的原理應出自對 Hyperplane separation theorem 的推導，該理論認為在 N 維空間中兩個不相交的凸集，如果這兩個集合都是閉集且至少其中一個是緊空間，那麼就會存在一個超平面可以將兩個集合分開，數學名詞較為複雜，如有興趣再深入了解即可。
應用到二維平面時，可以推導出兩個不相交的凸包之間會存在一條直線，這條直線可以將兩個凸包分開，稱作分離線，與該直線垂直的軸線稱作分離軸，兩個凸包在分離軸上的投影不會發生重疊。

實際來看些例子，從最簡單的兩個矩形開始，可以看到左邊不相交的矩形可以被線條 A 分開，分離軸是 X 軸，在 X 軸上的投影也沒有重疊，右邊相交的矩形則是在 XY 軸上的投影都產生了重疊。

![](/blog/images/SAT-AABB.jpg)

那麼問題來了，要如何找到分離軸？為何只要判斷 XY 軸上的投影都產生重疊就可以認定矩形相交了呢？

SAT 的理論是，以圖形上的所有邊為基準，邊的法向量作為分離軸，一一檢測投影是否產生重疊，如果全部都有重疊代表圖形發生相交，一旦有一個沒有發生重疊就認定圖形沒有發生相交，對上圖的矩形來說，八個邊產生的八個分離軸除掉重複的部分以後就只有 XY 軸。

那為何是以所有邊為基準？如何保證只要檢查完所有邊就可以認定圖形發生相交？

考慮一下圖形在發生相交前的狀態，滿足相交最小的條件就是兩個圖形互相接觸，也就是發生碰撞，可以分為三種情形，頂點碰到頂點、頂點碰到邊、邊碰到邊，想像兩個圖形不斷地靠近始終沒有發生碰撞，中間會存在一個無限小的空隙，此空隙間存在一條線段用來表示圖形間的距離，此線段必定垂直於其中一邊，自然可被視為分離軸，即使是頂點碰到頂點也是如此，可將頂點視為其中一邊的端點。

有了以上概念就可以知道，若兩個圖形沒有發生碰撞，至少存在一個分離軸垂直於其中一邊。

### 2.計算
上面的矩形計算上其實就跟 AABB 一樣，現在來看一下比較複雜的形狀。

***
參考資料
- *[Hyperplane separation theorem](https://en.wikipedia.org/wiki/Hyperplane_separation_theorem)*
- *[2D凸多边形碰撞检测算法（一） - SAT](https://zhuanlan.zhihu.com/p/176667175)*
- *[SAT (Separating Axis Theorem)](https://dyn4j.org/2010/01/sat/)*
- *[Intersection of Convex Objects: The Method of Separating Axes](https://www.geometrictools.com/Documentation/MethodOfSeparatingAxes.pdf)*
