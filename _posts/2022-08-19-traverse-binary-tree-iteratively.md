---
layout: post
title:  "[算法] 用迭代方式遍历树 Traverse Binary Tree Iteratively"
date: 2022-08-19 23:37:23-06:00
categories: algorithm
---
遍历树的常见方式有preorder(前序，根左右)，inorder(中序，左根右)，postorder(后序，左右根)。
递归(Recursive)的方法很容易写，我们看一下迭代(Iterative)的方法。

我们需要手动维护一个stack，每个节点进栈两次。
- 第一次进栈是为了处理此节点的左右子树
- 第二次进栈是为了处理此节点本身，最简单的情况就是记录或打印node.val

为了区分这两种状态，栈内我们存tuple，即`(node, are_children_visited)`。
当`are_children_visited`为`False`时，根据遍历要求(preorder/inorder/postorder)
决定入栈顺序。注意入栈顺序和处理顺序相反，后入栈的先处理。
- preorder根左右，入栈顺序是`node.right, node.left, node`
- inorder左根右，入栈顺序是`node.right, node, node.left`
- postorder左右根，入栈顺序是`node, node.right, node.left`

这是中序遍历的迭代写法，前序和后序除了入栈顺序，其他都相同。
{% highlight python linenos %}
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class Solution:
    def inorderTraversal(self, root: Optional[TreeNode]) -> List[int]:
        stack = [(root, False)]
        result = []
        while stack:
            node, are_children_visited = stack.pop()
            if node is None:
                continue
            if are_children_visited:
                result.append(node.val)
            else:
                stack.append((node.right, False))
                stack.append((node, True))
                stack.append((node.left, False))
        return result
{% endhighlight %}

左右子树入栈的时候判断None也可以，这样每次弹栈的元素就不用再判断None了。
前序遍历可以更简单，因为总是先处理当前节点，不需担心访问左右子树。

{% highlight python linenos %}
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class Solution:
    def preorderTraversal(self, root: Optional[TreeNode]) -> List[int]:
        stack = [root]
        result = []
        while stack:
            node = stack.pop()
            if node is None:
                continue
            result.append(node.val)
            stack.append(node.right)
            stack.append(node.left)
        return result
{% endhighlight %}
