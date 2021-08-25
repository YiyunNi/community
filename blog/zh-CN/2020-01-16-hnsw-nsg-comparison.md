---
id: 2020-01-16-hnsw-nsg-comparison.md
title: Milvus 揭秘系列（一）：向量索引算法HNSW和NSG的比较
author: 林孝君
date: 2021-07-30
desc: Open-source communities are creative and collaborative spaces. In that vein, the Milvus
banner: ../assets/blogCover.png
cover: ../assets/blogCover.png
tag: test1
---

# Milvus 揭秘系列（一）：向量索引算法 HNSW 和 NSG 的比较

> 作者：林孝君
>
> 日期：2020-01-16

随着机器学习、深度神经网络的不断发展，数据的向量化无处不在。而针对海量向量数据的搜索，无论是工业界还是学术界都做了大量的研究。本文主要讲解两个基于近邻图的向量搜索算法，并比较其适用场景。

这里不得不先提一个学术上的对应名词 Approximate Nearest Neighbor
Search (ANNS)，近似的最近邻搜索。之所以近似是由于精确的近邻搜索太过困难，研究随之转向了在精确性和搜索时间做取舍。由于精确的向量搜索在海量数据的场景下搜索时间过长，所以目前的常见做法，是在向量上建立近似搜索索引。

这里先介绍一下现在常用的索引类型以及它们的局限性。首先是基于树的算法，这里举例较为经典的 KD-tree。这种索引类型在向量维度稍大一些的情况下 (d>10)，索引性能会急剧下降甚至不如暴力搜索。再说说基于 LSH (locality-sensitive hashing) 的索引，如果想要取得高召回率，LSH 算法必须要建立大量的 Hash 表，这会使得索引大小膨胀数倍。不仅如此，树和 LSH 都属于空间切分类算法，此类算法有一个无法避免的缺陷，即为了提高搜索精度，只能增大搜索空间。图 1-A 描述了基于树的切分搜索，每个虚线分割出的区域是一个子树，如果搜索向量在子树的边缘时，算法需要搜索多个子树来获取结果。图 1-B 描述了基于 Hash 的切分搜索，虚线描述了每个独立的 hash 表，搜索面临的问题与基于树的搜索算法类似。所以空间切**分类算法在最坏的情况下需要扫描几乎整个数据集**，这在很多场景下显然是无法接受的。

![pic1](https://raw.githubusercontent.com/milvus-io/community/master/blog/assets/hnsw_nsg/pic1.png)

由于图数据结构有天生近邻关系的特性，在图上做最近邻搜索看来是一种不错的思路，所以近年来基于图的向量搜索算法也是一个研究热点。

## **近邻图(Proximity Graph)：最朴素的图算法**

近邻图的特性可以粗略理解成：构建一张图，每一个顶点连接着最近的 N 个顶点。它的搜索过程可见下图，Target 是待查询的向量。在搜索时，由于无法知道该从图的哪个区域开始搜索，所以我们选择任意一个顶点 S 出发。首先遍历 S 的邻居，找到距离与 Target 最近的 A 节点，将 A 设置为起始节点，再从 A 节点出发进行遍历，反复迭代，不断逼近，最后找到与 Target 距离最近的节点 A``时搜索结束。

![pic2](https://raw.githubusercontent.com/milvus-io/community/master/blog/assets/hnsw_nsg/pic2.png)

尽管思路很棒，但是基础的近邻图存在着非常多的问题，最为关键的就是搜索复杂度无法确定，孤岛效应难以解决，以及构建图的开销太高，复杂度达到了指数级别。由于这些原因，学术界在基础的近邻图上，对图的构建、度数的限制、边的裁选、图的连通性、节点的导向性等方面都做了许多改进。这里我们简单介绍下近年来较为热门的算法：HNSW 和 NSG。

## **HNSW**

HNSW 的前身是 NSW (Navigable-Small-World-Graph)。NSW 通过设计出一个具有导航性的图来解决近邻图发散搜索的问题，但其搜索复杂度仍然过高，达到多重对数的级别，并且整体性能极易被图的大小所影响。HNSW 则是着力于解决这个问题。作者借鉴了 SkipList 的思想，提出了 Hierarchical-NSW 的设想。简单来说，按照一定的规则把一张的图分成多张，越接近上层的图，平均度数越低，节点之间的距离越远；越接近下层的图平均度数越高，节点之间的距离也就越近（见下图）。

搜索从最上层开始，找到本层距离最近的节点之后进入下一层。下一层搜索的起始节点即是上一层的最近节点，往复循环，直至找到结果（见下图）。由于越是上层的图，节点越是稀少，平均度数也低，距离也远，所以可以通过非常小的代价提供了良好的搜索方向，通过这种方式减少大量没有价值的计算，减少了搜索算法复杂度。更进一步，如果把 HNSW 中节点的最大度数设为常数，这样可以获得一张搜索复杂度仅为 log(n) 的图。

![pic3](https://raw.githubusercontent.com/milvus-io/community/master/blog/assets/hnsw_nsg/pic3.png)

HNSW 利用多层图的优势，天生保证了图的连通性。并且由于每个新节点的加入都会被随机到任意一层中，这样做一定程度上避免了由于数据输入顺序从而改变图的分布，最后影响搜索路径的情况。

让我们来看看插入操作。新节点在插入到图中任意位置后，会建立 candidate pool，它用来保存所有搜索过的节点。首先将插入位置附近的节点加入到 pool 中，从 pool 中挑选出距离最近的节点，再从它出发寻找更近的节点，找到后将其加入到 pool 中（下图 2-A 到 2-B)，如果此节点都已搜索完毕，则挑选 pool 中的下一个节点进行搜索，不断循环 N 次，直至搜索到了 pool 的尾部，此时算法结束。在此过程中，如果 pool 满了但却发现了更近的节点，那么将它插入到 pool 中，剔除最远的节点，并且从新插入的位置处重新开始搜索（下图 2-B 到 2-C）。这样减少了边的数量，改善了局部图的导航性。

![pic4](https://raw.githubusercontent.com/milvus-io/community/master/blog/assets/hnsw_nsg/pic4.png)

## **NSG**

NSG 全写为 Navigating Spreading-out Graph。NSG 围绕四个方向来改进：图的连通性，减少出度，缩短搜索路径，缩减图的大小。具体是通过建立导航点 (Navigation Point)，特殊的选边策略, 深度遍历收回离散节点（Deep Traversal）等方法。

首先是 Navigation Point，在建图时，首先需要一张预先建立的 K-nearest-neighbor-graph (KNNG) 作为构图基准。随机选择一个点作为 Navigation Point，后续所有新插入的节点在选边时都会将 Navigation Point 加入候选。在建图过程中，逐渐会将子图都和 Navigation point 相连接，这样其他的节点只需保持很少的边即可，从而减少了图的大小。每次搜索从 Navigation Point 出发能够指向具体的子图，从而减少无效搜索，获得更好搜索性能。

NSG 使用的择边策略与 HNSW 类似，但是不同于 HNSW 只选择最短边为有效边，NSG 使用的择边策略如下图。以点 r 为例，当 r 与 p 建立连接时，以 r 和 p 为圆心，r 和 p 的距离为半径，分别做圆，如果两个圆的交集内没有其他与 p 相连接的点，则 r 与 p 相连（见图 3-B）。在连接点 s 时，由于以 s 和 p 距离为半径的交集圆内，已有点 r 与 p 相连，所以 s 和 p 不相连（见图 3-C）。下图中最终与点 p 相连的点只有 r, t 和 q（见图 3-A）。

NSG 这样做的原因是考虑到由于边数一多，整个图就会变得稠密，最后在搜索时会浪费大量算力。但是减少边的数量，带来的坏处也比较明显，最后的图会是一张稀疏图，会使得有些节点难以被搜索到。不仅如此，NSG 的边还是单向边。在如此激进的策略下，图的连通性就会产生问题，这时 NSG 选择使用深度遍历来将离群的节点收回到图中。通过以上步骤，图的建立就完成了。

![pic5](https://raw.githubusercontent.com/milvus-io/community/master/blog/assets/hnsw_nsg/pic5.png)

## **HNSW 和 NSG 的比较**

至此，我们大致了解了 HNSW 以及 NSG 两种算法的基本思路。HNSW 从结构入手，利用分层图提高图的导航性减少无效计算从而降低搜索时间，达到优化的目的。而 NSG 选择将图的整体度数控制在尽可能小的情况下，提高导航性，缩短搜索路径来提高搜索效率。

下面我们从内存占用，搜索速度，精度等方面来比较 HNSW 和 NSG。HNSW 由于多层图的结构以及连边策略，导致搜索时内存占用量会大于 NSG，在内存受限场景下选择 NSG 会更好。但是 NSG 在建图过程中无论内存占用还是耗时都大于 HNSW。此外 HNSW 还拥有目前 NSG 不支持的特性，即增量索引，虽然耗时巨大。对比其他的索引类型，无论 NSG 还是 HNSW 在搜索时间和精度两个方面，都有巨大优势。目前在 Milvus 内部已经实现了 NSG 算法，并将 KNNG 计算放到了 GPU 上进行从而极大地加快了 NSG 图的构建。未来 Milvus 还会集成 HNSW，以适配更广泛的场景。