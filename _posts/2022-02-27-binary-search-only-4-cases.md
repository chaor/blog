---
layout: post
title:  "[算法] 二分查找只有4种情况 binary search"
date: 2022-02-21 20:57:07-08:00
categories: algorithm
---
搞ACM竞赛的Xuehui大神告诉我们，二分查找只有4种情况。
- `mid`等于`target`时，向左走还是往右走
- 最后返回`left`还是`right`

来看这个从`[10, 20, 30, 30, 30, 30, 40, 50]`里找30的例子。
{% highlight python linenos %}
def binary_search(input_list, target):
    left, right = 0, len(input_list) - 1
    while left <= right:
        mid = (left + right) // 2
        # < or <= makes a difference
        if input_list[mid] <= target:
            left = mid + 1
        else:
            right = mid - 1
    # left or right makes a difference
    return left

print(binary_search([10, 20, 30, 30, 30, 30, 40, 50], 30))
{% endhighlight %}
结果是6。因为在第6行有等号的情况下，碰到30我们选择向右走。最后跳出while循环时，`right`在`left`左边，`right`指向30，`left`指向40。如果我们返回`left`就得到index为6，返回`right`就得到index为5。同理如果第6行不加等号，碰到30我们选择向左走，就会返回index 1或2。

List里面没有30的情况更简单:
{% highlight python %}
print(binary_search([10, 20, 25, 35, 40, 50], 30))
{% endhighlight %}
因为`target 30`不在list里，所以第6行的等号没用了，只需要看最后返回的是`left`还是`right`。跳出while循环时，`right`和`left`停在25和35。返回`left`会得到3，返回`right`会得到2。
