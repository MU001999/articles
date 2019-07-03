---
title: MTCNN理解
id: 353
categories:
  - Machine Learning
date: 2017-07-27 09:56:11
tags:
---

# Multi-task convolutional neural networks

许多地方参考了[MTCNN（Multi-task convolutional neural networks）人脸对齐
](http://blog.csdn.net/qq_14845119/article/details/52680940)这篇博文，在此表示感谢

### MTCNN检测流程
![](http://storage1.imgchr.com/AS2ND.png)
MTCNN总共由三个网络构成，分别是P-Net, R-Net, O-Net。<br>
首先分别概括三个网络的作用：
* Proposal Network (P-Net):
    * 获得了人脸区域的候选窗口和边界框的回归向量
    * 通过边界框回归，对候选窗口进行校准
    * 通过[NMS算法](http://www.jusot.com/?p=330)来合并高度重叠的候选框
* Refine Network (R-Net):
    * 通过边界框回归，对候选窗口进行校准
    * 通过[NMS算法](http://www.jusot.com/?p=330)来去除False-Positive区域
* Output Network (O-Net):
    * 比前两个网络多获得了五个地标(landmark)
    * 通过边界框回归，对候选窗口进行校准
    * 通过[NMS算法](http://www.jusot.com/?p=330)来去除False-Positive区域

在处理过程中，第一步是先将图像进行图像金字塔处理，<br>
然后将过程中各个大小的图像分别送入P-Net网络中，<br>
分别经过三个网络的作用，即返回出检测窗口和其中的特征点。<br>

### 训练
MTCNN特征描述子主要包含三个部分，即人脸/非人脸分类器，边界框回归和特征点(地标)定位。
#### 人脸分类
![](http://img.blog.csdn.net/20160927151710209)<br>
上式为人脸分类的交叉熵损失函数，pi为是人脸的概率，yidet为背景的真实标签。
#### 边界框回归
![](http://img.blog.csdn.net/20160927151729296)<br>
上式为基于欧氏距离计算的回归损失。其中，带尖的y为预测值，不带尖的y为实际的背景坐标。且y为一个四元组（左上角x，左上角y，长，宽）。
#### 特征点(地标)定位
![](http://img.blog.csdn.net/20160927151747812)<br>
计算预测的特征点位置和真实特征点的欧式距离，并最小化该距离。其中y为十元组.
#### 多个输入源的训练
![](http://img.blog.csdn.net/20160927151808203)<br>
整个训练学习过程即最小化上面的这个函数，N为训练样本数量，aj表示任务的重要性，bj为样本标签，Lj为上面的损失函数。

> 在训练过程中，为了取得更好的效果，作者每次只后向传播前70%样本的梯度，这样来保证传递的都是有效的数字。有点类似latent SVM，只是作者在实现上更加体现了深度学习的端到端。

在训练过程中,y尖和y的交并集IoU(Intersection-over-Union)比例:
* 0-0.3：非人脸
* 0.65-1.00：人脸
* 0.4-0.65：Part人脸
* 0.3-0.4：特征点

### References
1. http://blog.csdn.net/qq_14845119/article/details/52680940
1. https://kpzhang93.github.io/MTCNN_face_detection_alignment/index.html
