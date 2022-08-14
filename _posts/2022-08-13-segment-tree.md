---
layout: post
title:  "[算法] 线段树 Segment Tree"
date: 2022-08-13 22:41:07-07:00
categories: algorithm
---
线段树适合对一个range进行查询(比如求和，求最大值最小值)和更新某个值。树的叶节点是一个数，非叶节点是一个range。

对[307. Range Sum Query - Mutable](https://leetcode.com/problems/range-sum-query-mutable/)，
我们可以用数组来表示线段树。根结点下标为1，节点`i`的左右子节点下标为`2*i`和`2*i+1`。
数组的前n个位置留给非叶节点，后n个位置留给叶节点。查询range sum的时候，
对于L下标，它为奇数时parent是无法包含它的，因此需要单独加上这个值，并让L=L+1。
同理，R下标为偶数时parent无法包含，需要单独加上这个值，并让R=R-1。

{% highlight python linenos %}
class NumArray:

    def __init__(self, nums: List[int]):
        self.n = len(nums)
        self.tree = [0] * self.n + nums
        for i in range(self.n - 1, 0, -1):
            self.tree[i] = self.tree[2 * i] + self.tree[2 * i + 1]

    def update(self, index: int, val: int) -> None:
        index += self.n
        diff = val - self.tree[index]
        while index:
            self.tree[index] += diff
            index //= 2
            
    def sumRange(self, left: int, right: int) -> int:
        L, R = left + self.n, right + self.n
        sum_ = 0
        while L <= R:
            if L % 2 == 1:
                sum_ += self.tree[L]
                L += 1
            if R % 2 == 0:
                sum_ += self.tree[R]
                R -= 1
            L //= 2
            R //= 2
        return sum_
{% endhighlight %}

概念上是二叉树，但我们并没有定义TreeNode。如果定义TreeNode，除了`left`和`right`这两个field，
还需要`start`, `end`和`sum`这几个field。递归终止的条件是start==end，但代码量有点多。
