---
title: 第七部分 - B树
date: 2017-09-23
---

SQLite 的表和索引的数据结构都采用了B树（B-Tree），因此B树是一个非常重要的概念。在本章中，我们将只介绍B树这种数据结构，因此不会有任何代码。

为什么说树是一个非常适合数据库的数据结构呢？

- 查找一个特定的值非常快（只需要对数时间）
- 插入/删除一个已经找到的值是非常快的（重新平衡的时间很稳定）
- 范围遍历非常快（相对哈希表而言）

B树不同于二叉树（“B”可能表示的是发明者的名字，也可以表示“balanced” *注：「平衡的」*）。下面是一个B树的例子：

{% include image.html url="assets/images/B-tree.png" description="example B-Tree (https://en.wikipedia.org/wiki/File:B-tree.svg)" %}

和二叉树不同，B树的每一个节点可以拥有超过两个子节点。每个节点可以拥有最多 m 个子节点，m 又被叫做树的“阶”。为了使树尽可能的接近平衡，我们还说节点必须拥有至少 m/2 （向上取整）个子节点。

例外：
- 叶子节点没有子节点
- 根节点的子节点数可以少于 m，但必须不小于 2
- 如果根节点就是叶子节点（唯一的节点），那么它可以没有子节点

上图所示是一个B树，在 SQLite 中被用于存储索引，而为了存储表数据，SQLite 用了B树的一种变体，我们称之为 B+ 树。

|                               | B-tree         | B+ tree             |
|-------------------------------|----------------|---------------------|
| 发音                           | "Bee Tree"     | "Bee Plus Tree"     |
| 用于存储                       | 索引            | 表数据               |
| 内部节点是否存储键               | 是             | 是                   |
| 内部节点是否存储值               | 是             | 否                   |
| 每个节点的子节点数量              | 少             | 多                |
| 内部节点和叶子节点结构对比         | 相同           | 不同                 |

在我们开始实现索引之前，我将只介绍 B+ 树，但我会将它称之为B-树（B-Tree）或者B树（btree）。

“内部”节点指那些拥有子节点的节点。内部节点和叶子节点的结构是不一样的：

| 对于一棵 m 阶树...       | 内部节点                       | 叶子节点             |
|------------------------|-------------------------------|---------------------|
| 存储的数据               | 键和指向子节点的指针             | 键和值               |
| 键的数量                | 最多 m-1                       | 尽可能的多            |
| 指针的数量               | 键数量 + 1                     | 0                   |
| 值的数量                | 0                             | 等于键的数量          |
| 键的用途                | 用于路由                        | 与值匹配             |
| 是否存储值？             | 否                            | 是                  |

Let's work through an example to see how a B-tree grows as you insert elements into it. To keep things simple, the tree will be order 3. That means:

- up to 3 children per internal node
- up to 2 keys per internal node
- at least 2 children per internal node
- at least 1 key per internal node

An empty B-tree has a single node: the root node. The root node starts as a leaf node with zero key/value pairs:

{% include image.html url="assets/images/btree1.png" description="empty btree" %}

If we insert a couple key/value pairs, they are stored in the leaf node in sorted order.

{% include image.html url="assets/images/btree2.png" description="one-node btree" %}

Let's say that the capacity of a leaf node is two key/value pairs. When we insert another, we have to split the leaf node and put half the pairs in each node. Both nodes become children of a new internal node which will now be the root node.

{% include image.html url="assets/images/btree3.png" description="two-level btree" %}

The internal node has 1 key and 2 pointers to child nodes. If we want to look up a key that is less than or equal to 5, we look in the left child. If we want to look up a key greater than 5, we look in the right child.

Now let's insert the key "2". First we look up which leaf node it would be in if it was present, and we arrive at the left leaf node. The node is full, so we split the leaf node and create a new entry in the parent node.

{% include image.html url="assets/images/btree4.png" description="four-node btree" %}

Let's keep adding keys. 18 and 21. We get to the point where we have to split again, but there's no room in the parent node for another key/pointer pair.

{% include image.html url="assets/images/btree5.png" description="no room in internal node" %}

The solution is to split the root node into two internal nodes, then create new root node to be their parent.

{% include image.html url="assets/images/btree6.png" description="three-level btree" %}

The depth of the tree only increases when we split the root node. Every leaf node has the same depth and close to the same number of key/value pairs, so the tree remains balanced and quick to search.

I'm going to hold off on discussion of deleting keys from the tree until after we've implemented insertion.

When we implement this data structure, each node will correspond to one page. The root node will exist in page 0. Child pointers will simply be the page number that contains the child node.

Next time, we start implementing the btree!
