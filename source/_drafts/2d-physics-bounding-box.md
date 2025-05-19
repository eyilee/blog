---
title: 2D 碰撞檢測包圍盒
categories: 2D Physics
tags:
---

在碰撞檢測中可以分為兩個階段 Broad-Phase 和 Narrow-Phase，在 Broad-Phase 中先進行粗略的篩選可能碰撞的物體，之後才會在 Narrow-Phase 做精細的計算判斷兩物體是否產生碰撞。

為何需要分為兩個階段進行？假設場景中有兩顆球在移動，如果要判斷兩球是否產生碰撞，一顆球只需計算和另一顆球的碰撞，若有三顆球則變為兩次，以此類推若有 N 顆球則每顆球需要計算 N-1 次，計算複雜度為 $\mathcal{O(N^2)}$，這樣的計算量顯然是不能使用的，因此需要 Broad-Phase 先對物體進行篩選。

在 Broad-Phase 通常會先將物體簡化為包圍盒，常見的包圍盒有 AABB(Axis-Aligned Bounding Box)、OBB(Oriented Bounding Box) 等，同時使用四叉樹(QuadTree)、BVH(Bounding volume hierarchy) 等技術將物體所在的空間作分割達到減少計算量的目的。

本文將介紹 AABB、OBB 這兩種包圍盒。

<!-- more -->
### 1. AABB(Axis-Aligned Bounding Box)
AABB 顧名思義就是按照和座標軸對齊的包圍盒，四個邊都會和座標軸對齊，因此只需要對角的兩個頂點就可以記錄一個 AABB 包圍盒，如下圖。

![](/blog/images/AABB.jpg)
> 藍色三角形的 AABB 包圍盒就是紅色矩形。

兩個 AABB 包圍盒的碰撞檢測也很簡單，兩個包圍盒在每一個座標軸上的投影都會產生重疊，如下圖。

![](/blog/images/AABB-collision.jpg)
> 橘色線段即兩包圍盒在座標軸上重疊的部分。

假設有一包圍盒 $A$ 的頂點分別是 $(x_1, y_1) ,\space (x_2, y_2)$ 且 $x_1 < x_2 ,\space y_1 < y_2$，和另一包圍盒 $B$ 發生碰撞，包圍盒 $B$ 則會有一頂點 $(x, y)$ 在包圍盒 $A$ 的內部，滿足以下條件 $x_1 \eqslantless x \eqslantless x_2 ,\space y_1 \eqslantless y \eqslantless y_2$。

AABB 在碰撞檢測的計算上十分快速，但有一個缺點，因為四個邊都是和座標軸對齊的，一旦圖形產生旋轉，頂點無法直接同時旋轉使用，必須重新計算新的頂點。

### 2. OBB(Oriented Bounding Box)


***
參考資料
- *[Bounding volumee](https://en.wikipedia.org/wiki/Bounding_volume)*
- *[游戏物理引擎(二) 碰撞检测之Broad-Phase](https://zhuanlan.zhihu.com/p/113415779)*
- *[现代游戏物理引擎入门(三)——碰撞检测(上)](https://zhuanlan.zhihu.com/p/396719279)*
