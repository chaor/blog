---
layout: post
title: "[算法] 并查集 Disjoint set"
date: 2022-02-07 22:38:29-08:00
categories: algorithm
---
给定一些元素，并查集(disjoint set, union-find set)可以
- 查询一个元素属于哪个集合(find)
- 合并两个不相交集合(union)。

我们可以用并查集来看图中有没有环。比如规定同一集合中的顶点为互相连通的，那我们把下图按边来union，边1-2, 1-3, 2-4可以把顶点1,2,3,4并入同一个集合。

    1 -- 2
    ｜   ｜
    ｜   ｜
    3 -- 4

检查边3-4时，发现顶点3和4已在同一个集合，也就是除了边3-4外，顶点3和顶点4已经联通，于是知道图中有环。

并查集的实现使用一个一维`parent`数组，长度为元素(即节点)个数。`parent[x]`表示节点`x`的父节点。一开始每个元素都是独立的集合，互不相交，所以初始状态是`parent[x]=x`。随着不断进行`Union(x, y)`操作，这些节点会形成一棵棵树。Find操作检查一个元素在哪棵树，即它的根节点`root`是谁，所以函数名叫`find_root`更好。

{% highlight python %}
def find_root(x: int, parent: List[int]) -> int:
    root = x
    while parent[root] != root:
        root = parent[root]
    # 这里进行路经压缩。因为如果树退化成一条链，单次查询时间很长。
    parent[x] = root
    return root
{% endhighlight %}

`union`函数就是找出`x`和`y`的根结点，如果相同就直接返回`0`，不相同就把一个根节点的`parent`设为另一个根节点，于是两棵树合并成一棵树，返回`1`，这个返回值记录我们进行了多少次有效的`union`操作。盲目合并两棵树可能增大树的高度，增加下次`find_root`的开销。所以我们可以用另一个`rank`(也就是秩)数组来记录树的高度，初始值设为1。只有在两棵树`rank`相同时，我们才将新的树`rank` + 1。

{% highlight python %}
def union(x: int, y: int, parent: List[int], rank: List[int]) -> int:
    x_root, y_root = find_root(x), find_root(y)
    if x_root == y_root:
        return 0

    if rank[x_root] > rank[y_root]:
        parent[y_root] = x_root
    elif rank[x_root] < rank[y_root]:
        parent[x_root] = [y_root]
    else:
        # 两边rank一样，随便合并
        rank[y_root] += 1
        parent[x_root] = y_root
    return 1
{% endhighlight %}

[200. Number of Islands](https://leetcode.com/problems/number-of-islands/)给一个`m`x`n` 2维数组，1表示岛屿，0表示水，周围区域也都是水。水平和竖直方向相连的1构成一个岛屿，求岛屿数目。我们可以扫一遍这个2维数组，碰到1就看左边和上边是否有1，是的话就`union`操作。通过辅助函数把`(x, y)`坐标变成一维。岛屿数 = 1的数目 - 有效`union`操作次数。

{% highlight python %}
class Solution:
    def get_index(self, i: int, j: int) -> int:
        return self.n * i + j
    
    def find_root(self, x: int) -> int:
        root = x
        while self.parent[root] != root:
            root = self.parent[root]
        self.parent[x] = root
        return root
    
    def union(self, x: int, y: int) -> int:
        x_root, y_root = self.find_root(x), self.find_root(y)
        if x_root == y_root:
            return 0

        if self.rank[x_root] > self.rank[y_root]:
            self.parent[y_root] = x_root
        elif self.rank[x_root] < self.rank[y_root]:
            self.parent[x_root] = y_root
        else:
            self.rank[y_root] += 1
            self.parent[x_root] = y_root
        return 1
    
    def numIslands(self, grid: List[List[str]]) -> int:
        self.m = len(grid)
        self.n = len(grid[0])
        self.parent = list(range(self.m * self.n))
        self.rank = [1] * (self.m * self.n)
   
        union_count = 0
        number_1_count = 0
        for i, row in enumerate(grid):
            for j, value in enumerate(row):
                if value == '0':
                    continue
                number_1_count += 1
                if i - 1 >= 0 and grid[i - 1][j] == '1':
                    union_count += self.union(self.get_index(i, j), self.get_index(i - 1, j))
                if j - 1 >= 0 and grid[i][j - 1] == '1':
                    union_count += self.union(self.get_index(i, j), self.get_index(i, j - 1))
        return number_1_count - union_count
{% endhighlight %}

幸甚至哉，歌以咏志。
