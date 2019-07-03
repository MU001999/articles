---
title: 非极大值抑制算法（Python实现）
id: 330
categories:
  - Machine Learning
date: 2017-07-21 16:48:02
tags:
---

# 非极大值抑制算法（Non-maximum suppression, NMS）

### 算法原理
非极大值抑制算法的本质是搜索局部极大值，抑制非极大值元素。

### 算法用途
如在物体检测中可以通过应用NMS算法来消除多余的交叉重复的窗口，使在同一物体的多个检测窗口中保留下得分最高的窗口。
<br>NMS算法亦可用于视频跟踪/数据挖掘/3D重建以及文理分析等。

### 算法实现思路
首先迭代所有的点，迭代每一个点的时候判断该点是否符合局部最大值的条件。

### NMS算法在三邻域情况下的实现
三邻域情况下的NMS即判断一维数组array中的元素array[i]是否大于其左邻元素array[i-1]和右邻元素array[i+1]，具体实现如下图（Python表示）：
```python
import numpy as np

array = [0] + np.random.randint(100, size=10).tolist() + [0]
keep = []
i = 1

while i <= 10:
    if array[i] > array[i+1]:
        if array[i] > array[i-1]:
            keep.append(array[i])
    else:
        i += 1
        while i <= 10 and array[i] <= array[i+1]:
            i += 1
        if i <= 10:
            keep.append(array[i])
    i += 2
```

### NMS算法应用于人脸检测窗口选择的实现（Python实现）
```python
import numpy as np


def nms(rects, threshold):
    x1, y1, x2, y2, scores = rects[:, 0], rects[:, 1], rects[:, 2], rects[:, 3], rects[:, 4]
    
    areas = (x2 - x1 + 1) * (y2 - y1 + 1)
    order = scores.argsort()[::-1]
    keep = []
    while order.size > 0:
        i = order[0]
        keep.append(i)
        xx1 = np.maximum(x1[i], x1[order[1:]])
        yy1 = np.maximum(y1[i], y1[order[1:]])
        xx2 = np.minimum(x2[i], x2[order[1:]])
        yy2 = np.minimum(y2[i], y2[order[1:]])
        
        inter = np.maximun(0.0, xx2 - xx1 + 1) * np.maximum(0.0, yy2 - yy1 + 1)
        iou = inter / (areas[i] + areas[order[1:]] - inter)
        indexs = np.where(iou <= threshold)[0]
        order = order[indexs + 1]
        
    return keep
```