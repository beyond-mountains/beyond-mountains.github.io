---
title: Join Order
date: 2024-04-14 00:00:00 +0800
categories: [Database System, Query Optimization]
tags: [Join order]     # TAG names should always be lowercase
math: true
---

# Join Order

在关系型数据库中，Join（或连接）操作是查询执行时最重要的操作之一，也是用户经常用的操作之一。但是表的连接通常需要花费较高的代价，当有多个表进行连接时，选择一个代价较小的连接顺序是一个值得考虑的问题。

下面先介绍一下连接顺序，先考虑一个 SQL 语句

```sql
select * from A, B, C, D;
```

我们将连接顺序抽象成 二叉树 的形式

<img src="/assets/img/PictureJoinOrder/LeftTree.jpg" style="zoom:50%;" />

表A 和 表B 连接，AB的中间结果与C连接，ABC的中间结果与D连接，形成最终结果。

<img src="/assets/img/PictureJoinOrder/SomeJoinOrder.jpg" style="zoom:50%;" />

但是连接的顺序并不一定按照 A——B——C——D，可以对 A B C D 进行排序，对于这 4 个表的情况，共有 $ 4! = 24 $ 种排列方式。

上面展示的 **连接树** 都有一个相同的特点，其子树都是向左延申，每个节点的右孩子是叶子结点，这种类型的树称为 **左深树** 。当然，有左深树，就有右深树。

如果我们不只是将连接顺序表示成 左深树 ，那么会有其他的树形结构

<img src="/assets/img/PictureJoinOrder/JoinOrderTree.jpg" style="zoom:50%;" />

对于四个表连接的情况，可以有 5 种树型结构。

对于 n 个表树形状的所有数目 T(n) 的递归公式为：

$$ T(1) = 1 $$

$$ T(n) = \sum \limits_{i=1} \limits^{n-1} T(i) T(n-i) $$

$T(n)$ 的前几个值为

| n    | 1    | 2    | 3    | 4    | 5    | 6    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| T(n) | 1    | 1    | 2    | 5    | 14   | 42   |

如果将一个树型结构下的连接顺序乘以不同的树型结构，$T(n) * n!$ 表示 n 个表相连接时，所有可能的连接顺序。

等价于公式 $ \frac{(2*(N-1))!}{(N-1)} $ 。



在一些方法中，只考虑以左深树作为可能的连接顺序。这有两个优点：

* 减少了搜索空间。所过考虑所有的树型结构，搜素空间会很大。
* 用于连接的左深树可以和通用的连接算法很好的交互。



如果这两个表没有相同的属性或者没有加限制条件，那么这两个表的连接称为 **交叉连接**。也就是两个表的笛卡尔积，这样会产生大量的中间结果。语句`select * from A, B, C, D;`就是四个表的交叉连接。在实际操作中，应尽量避免交叉连接的使用。

如果两个表有相同的属性，且SQL语句中有限制条件，那么这两个表可以根据这个属性进行连接。

<img src="/assets/img/PictureJoinOrder/Relations.jpg" style="zoom:50%;" />

上图中有三种不同的关系结构，结点之间的连线表示这两个表有相同的属性。

如果是第一种链式结构，分析 SQL 语句：

```sql
select * from A, B, C, D 
where A.attr1 = B.attr1
and B.attr2 = C.attr2
and C.attr3 = D.attr3;
```

可以给出 5 种连接顺序：

<img src="/assets/img/PictureJoinOrder/JoinOrderTree.jpg" style="zoom:50%;" />

对于第二种星型结构，有 6 种连接顺序；对于第三种环形结构，有 10 种连接顺序。



更进一步讲：

DBMS解决采用哪种连接顺序问题时，会比较复杂。

举一个例子：

* N 表 join 时，表的连接顺序有 N! 种组合，N=5时，N!= 120。

* 每种 join 可以选择多种连接算法，如 NLJ、BNL、HJ、MJ，PWJ 等。

  NLJ：Nested-Loop Join（嵌套循环连接） 。

  BNL：Block Nested-Loop Join（基于块的嵌套循环连接）。

  HJ：Hash Join（哈希连接）。

  MJ：merge Join（合并连接）。

  PWJ：Partition-Wise Join（分区连接）。

* 每张表还要决策是否走索引，最多 2^N 种组合

单这几样算下来，N=5 时大约有 32 * 10 * 120 种路径选择。计算 3w 种组合的代价，代价非常昂贵。

通常来说，5 表 join 实际只会产生一个非常有限的 access path，详细多少依赖于各个系统的实现。

优化器模块很大一部分精力就是来优化 access path 计算，这是一个数学上有解但工程上复杂的问题。