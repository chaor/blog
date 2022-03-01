---
layout: post
title:  "[算法] 动态规划 dynamic programming"
date: 2022-03-01 01:09:05-08:00
categories: algorithm
---
动态规划题目需要找准状态转移方程。需要注意的是`dp[i]`可能由`dp[i-1]`得来，也可能由`dp[1]`, `dp[2]`, `dp[3]`...`dp[i - 1]`共同得来。

比如[2188. Minimum Time to Finish the Race](https://leetcode.com/problems/minimum-time-to-finish-the-race/)，给一组轮胎，每组轮胎有`f`和`r`，跑第`lap`圈需要的时间为`f * r ^ (lap - 1)`，换轮胎的时间是`changeTime`，求跑`numLaps`圈的最小时间。轮胎是无限的，每跑完一圈都可以换任意一组轮胎。如果不换轮胎，跑圈时间指数级增加。如果换轮胎，多一个`changeTime`的时间。

本题不难想到用`dp[i]`来表示跑`i`圈的最小时间。难的是以下两点：
- `dp[3]`表示跑三圈的最小时间，这三圈可以是任意连着的三圈，比如第5-7圈。这连着的三圈中间可能换过轮胎，也可能没换过。`dp[3]`的值是确定的，因为第5-7圈可以完全复制第1-3圈的轮胎选择。
- `dp[i]`和`dp[1]`，`dp[2]` ... `dp[i - 1]`都有关系，因为换轮胎可以发生在任意圈数。`dp[7]`可能来自`dp[1]` + `changeTime` + `dp[6]`，也可能来自`dp[3]` + `changeTime` + `dp[4]`

因为`r >= 2`而`changeTime <= 100000`，2 ** 16 = 65536，所以前17圈是可能不换轮胎的，第18圈一定要换轮胎才能让时间更小。我们可以用不换轮胎的情况作为`dp[1]` ~ `dp[17]`的初始值。

{% highlight python linenos %}
import math

class Solution:
    def minimumFinishTime(self, tires: List[List[int]], changeTime: int, numLaps: int) -> int:
        dp = [0] + [math.inf] * numLaps
        for f, r in tires:
            current_time = f
            total_time = f
            for i in range(1, min(17, numLaps + 1)):
                dp[i] = min(dp[i], total_time)
                current_time *= r
                total_time += current_time
           
        for i in range(2, numLaps + 1):
            dp[i] = min(dp[i], min(dp[j] + changeTime + dp[i - j] for j in range(1, i // 2 + 1)))

        return dp[numLaps]
{% endhighlight %}
