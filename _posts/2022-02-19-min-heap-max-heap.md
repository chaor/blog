---
layout: post
title:  "[算法] 最大堆最小堆 max-heap min-heap"
date: 2022-02-19 22:59:01-08:00
categories: algorithm
---
Python的`heapq`模块只支持最小堆，保证堆顶(即二叉树的root)`heap[0]`的值为最小。如果是整数，那么乘以`-1`就变成了最大堆。如果是字母，要求从大到小排序，可以转成整数，把`-ord(char)`放入堆。另外Python有灵活的tuple，我们可以往堆里放tuple，让这些tuple来满足我们对元素的排序要求，无需像其他语言那样提供自定义的comparator函数。

一般需要用到`heapify` `heappush` `heappop`三个函数和先push再pop的`heappushpop`函数。初始化完list之后，用`heapify`把list变成堆。
{% highlight python %}
In [1]: import heapq

In [2]: min_heap = [20, 10, 40, 50, 30]

In [3]: heapq.heapify(min_heap)

In [4]: heapq.heappush(min_heap, 12)

In [5]: heapq.heappop(min_heap)
Out[5]: 10

In [6]: heapq.heappop(min_heap)
Out[6]: 12

In [7]: heapq.heappushpop(min_heap, 60)
Out[7]: 20
{% endhighlight %}

比如[2182. Construct String With Repeat Limit](https://leetcode.com/problems/construct-string-with-repeat-limit/)，给定字符串`s`，要求对其中字母重新排序，拿到字典序最大的字符串,同一个字母不能连续出现`repeatLimit`次。这种元素需要排序，且中途有些元素个数可能消耗为0的情况适合用堆，因为手动维护一个计数结构要记得删掉个数为0的元素，而用堆就很方便了，元素个数为0时，不再进堆就可以了。这个题还需要记录当前字母repeat的次数，不能超过`repeatLimit`。一次处理一个字母就好了，如果考虑每次处理`repeatLimit`个字母反而更复杂。

{% highlight python %}
import heapq
from collections import Counter


class Solution:
    def repeatLimitedString(self, s: str, repeatLimit: int) -> str:
        # python heapq only has min-heap, so we use -1 * ord(letter) to make it a max-heap
        max_heap = [(-ord(letter), count) for letter, count in Counter(s).items()]
        heapq.heapify(max_heap)
        answer_list = []
        repeat = 0

        while max_heap:
            letter_1, count_1 = heapq.heappop(max_heap)

            # when repeat == repeatLimit
            if answer_list and answer_list[-1] == letter_1 and repeat == repeatLimit:
                if not max_heap:
                    break
                letter_2, count_2 = heapq.heappop(max_heap)
                answer_list.append(letter_2)
                if count_2 - 1:
                    heapq.heappush(max_heap, (letter_2, count_2 - 1))
                heapq.heappush(max_heap, (letter_1, count_1))
                continue

            # when repeat < repeatLimit
            if answer_list and answer_list[-1] == letter_1:
                repeat += 1
            else:
                repeat = 1
            answer_list.append(letter_1)
            if count_1 - 1:
                heapq.heappush(max_heap, (letter_1, count_1 - 1))

        return ''.join(chr(-x) for x in answer_list)
{% endhighlight %}

最小堆另一个经典题目是[23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)，这里Python的tuple优势体现得淋漓尽致，我们可以把tuple `(value, sub_list)`放入最小堆。因为`value`可能有重复，所以再加一个sub_list_index `(value, index, sub_list)`。

{% highlight python %}
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
import heapq

class Solution:
    def mergeKLists(self, lists: List[Optional[ListNode]]) -> Optional[ListNode]:
        dummy_head = ListNode(0)
        current_node = dummy_head
        min_heap = [(sub_list.val, index, sub_list) for index, sub_list in enumerate(lists) if sub_list is not None]
        heapq.heapify(min_heap)

        while min_heap:
            value, index, sub_list = heapq.heappop(min_heap)
            current_node.next = ListNode(value)
            current_node = current_node.next
            if sub_list.next is not None:
                heapq.heappush(min_heap, (sub_list.next.val, index, sub_list.next))
        return dummy_head.next
{% endhighlight %}
