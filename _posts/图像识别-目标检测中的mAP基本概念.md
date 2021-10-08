---
title: 图像识别-目标检测中的mAP基本概念
tags: 机器学习
categories: 机器学习
abbrlink: 7acbd551
date: 2021-02-19 19:31:25
---

mAP的计算是做图像识别，特别是目标检测的时候，非常常见的知识，关于mAP的基本概念，外国的朋友已经做了非常详细的描述：

[点击进入查看](https://datascience.stackexchange.com/questions/25119/how-to-calculate-map-for-detection-task-for-the-pascal-voc-challenge)

简单来说，mAP=mean Average Precision, 即各类别AP的平均值，如果要计算出这个值，我们需要如下的数据作为支撑：

* AP: PR曲线下面积，后文会详细讲解
* PR曲线: Precision-Recall曲线
* Precision: TP / (TP + FP)
* Recall: TP / (TP + FN)
* TP: IoU>0.5的检测框数量（同一Ground Truth只计算一次）
* FP: IoU<=0.5的检测框，或者是检测到同一个GT的多余检测框的数量
* FN: 没有检测到的GT的数量

这里面由引申出一个知识点：IoU，IOU = 两个矩形交集的面积/两个矩形的并集面积，直观来说如下图所示：

![a](00001.png)
![b](00002.png)

好了，回到mAP上来说，可以知道，要计算mAP必须先绘出各类别PR曲线，计算出AP，而如何采样PR曲线，VOC采用过两种不同方法。


* 在VOC2010以前，只需要选取当Recall >= 0, 0.1, 0.2, ..., 1共11个点时的Precision最大值，然后AP就是这11个Precision的平均值。
* 在VOC2010及以后，需要针对每一个不同的Recall值（包括0和1），选取其大于等于这些Recall值时的Precision最大值，然后计算PR曲线下面积作为AP值。

一个计算示例：

假设，对于Aeroplane类别，我们网络有以下输出(BB表示BoundingBox序号，IoU>0.5时GT=1)：

```
BB  | confidence | GT
----------------------
BB1 |  0.9       | 1
----------------------
BB2 |  0.9       | 1
----------------------
BB1 |  0.8       | 1
----------------------
BB3 |  0.7       | 0
----------------------
BB4 |  0.7       | 0
----------------------
BB5 |  0.7       | 1
----------------------
BB6 |  0.7       | 0
----------------------
BB7 |  0.7       | 0
----------------------
BB8 |  0.7       | 1
----------------------
BB9 |  0.7       | 1
----------------------
```

因此，我们有 TP=5 (BB1, BB2, BB5, BB8, BB9), FP=5 (重复检测到的BB1也算FP)。除了表里检测到的5个GT以外，我们还有2个GT没被检测到，因此: FN = 2. 这时我们就可以按照Confidence的顺序给出各处的PR值，如下：

```
rank=1  precision=1.00 and recall=0.14
----------
rank=2  precision=1.00 and recall=0.29
----------
rank=3  precision=0.66 and recall=0.29
----------
rank=4  precision=0.50 and recall=0.29
----------
rank=5  precision=0.40 and recall=0.29
----------
rank=6  precision=0.50 and recall=0.43
----------
rank=7  precision=0.43 and recall=0.43
----------
rank=8  precision=0.38 and recall=0.43
----------
rank=9  precision=0.44 and recall=0.57
----------
rank=10 precision=0.50 and recall=0.71
----------
```

对于上述PR值，如果我们采用：

* VOC2010之前的方法，我们选取Recall >= 0, 0.1, ..., 1的11处Percision的最大值：1, 1, 1, 0.5, 0.5, 0.5, 0.5, 0.5, 0, 0, 0。此时Aeroplane类别的 AP = 5.5 / 11 = 0.5
* VOC2010及以后的方法，对于Recall >= 0, 0.14, 0.29, 0.43, 0.57, 0.71, 1，我们选取此时Percision的最大值：1, 1, 1, 0.5, 0.5, 0.5, 0。此时Aeroplane类别的 AP = (0.14-0)*1 + (0.29-0.14)*1 + (0.43-0.29)*0.5 + (0.57-0.43)*0.5 + (0.71-0.57)*0.5 + (1-0.71)*0 = 0.5


mAP就是对每一个类别都计算出AP然后再计算AP平均值就好了

