---
title: '[LeetCode] Largest Rectangle in Histogram'
id: 276
categories: 
  - Algorithm
date: 2017-05-30 02:17:37
tags:
---

### 84. Largest Rectangle in Histogram

Given n non-negative integers representing the histogram's bar height where the width of each bar is 1, find the area of largest rectangle in the histogram.<br>
![](https://leetcode.com/static/images/problemset/histogram.png)
<br>

Above is a histogram where width of each bar is 1, given height = `[2,1,5,6,2,3]`.
<br>
![](https://leetcode.com/static/images/problemset/histogram_area.png)
<br>
The largest rectangle is shown in the shaded area, which has area = `10` unit.
<br>
For example,<br>
Given heights = `[2,1,5,6,2,3]`,<br>
return `10`.

###### Solution

```python
class Solution(object):
    def largestRectangleArea(self, heights):
        stack = [-1]
        heights.append(0)
        res = 0
        for i in xrange(len(heights)):
            while heights[i] < heights[stack[-1]]:
                h = heights[stack.pop()]
                w = i - stack[-1] - 1
                res = max(res, h*w)
            stack.append(i)
        return res
```

### 解析：

stack中存储已遍历方块中与未遍历方块连通的最大高度（由小于左边方块高度的方块的高度决定）
