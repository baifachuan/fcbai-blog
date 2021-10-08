---
title: 使用Darknet框架中的Yolo模型使用自己的数据集进行训练
tags: 机器学习
categories: 机器学习
abbrlink: 53fd00f1
date: 2021-02-20 13:39:26
---

该模型使用的是YoloV4，Darknet代码地址为： https://github.com/AlexeyAB/darknet.git

基础知识：

Yolo中输出图像的计算方法：
```
output = （input-filter_size+2*padding）/（stride）+ 1
```

yolo层的前一层filter计算方法：
```
filters = (classes + 5) * 预测框的个数
```
预测框的个数也就是我们的分类的种类，特征融合一般操作：

* Route from previous layer
* Conv it for 1~ times
* Do upsample
* Route from the corresponding layer with same size of feature map
* Continue

当训练集制作好了后，就可以使用Draknet训练自己的数据集了，简单来说，步骤如下：

* 下载预训练权重yolov4.conv.137，放到darknet目录下，该预训练权重下载地址：https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v3_optimal/yolov4.conv.137
* 进入darknet/cfg目录下，复制yolov4-custom.cfg，名字改为yolov4-cat.cfg，并打开该文件，进行下面的6处修改：
    1. yolov4-cat.cfg文件第1-7行如下：
        ```
        [net]
        # Testing
        #batch=1
        #subdivisions=1
        # Training
        batch=64
        subdivisions=16
        ```
    注意：由于是进行训练，这里不需要修改。训练过程中可能出现CUDA out of memory的提示，可将这里的subdivisions增大，如32或64，但是数值越大耗时越长，因此需要权衡一下；

    2. yolov4-cat.cfg文件第8-9行将608修改为416：
        ```
        width=416
        height=416
        ```
    注意：这里也可不改，如果原始的数值608可能会导致CUDA out of memory的提示，而且这里的数值必须是32的倍数，这里也是数值越大耗时越长；

    3. 第20行的参数max_batches也要修改，原始值为500500，max_batches = classes*2000，但是max_batches不要低于训练的图片张数，这里只训练19类，因此max_batches = 38000；

    4. 第22行的参数steps=1600,1800，这两个数值分别为max_batches的80%和90%；

    5. 继续修改yolov4-cat.cfg文件，按Ctrl+F键，搜索“classes”，一共有3处，先定位到第一处，将classes=80改为classes=18，并将classes前面最近的filters修改为72，计算由来（classes+5）*3=72；

    6. 继续修改yolov4-cat.cfg文件，按照上面的步骤同样修改第二处和第三处的classes；
* 进入darknet/data文件夹下，创建名称为cat.names的文件（参考该文件夹voc.names文件的写法），在cat.names文件中添加类别名称；
* 进入darknet/cfg文件夹下，创建名称为cat.data的文件，在该文件中添加相关内容，一共5行，参考示例voc.data文件，类别改为19。
    ```
    classes= 19
    train  = /Users/fcbai/Downloads/yolov4-pytorch/data/train.txt
    valid  = /Users/fcbai/Downloads/yolov4-pytorch/data/val.txt
    names = /Users/fcbai/Downloads/darknet/data/vanke_coco_classes.txt
    backup = backup/
    ```
* 进入darknet目录下，右键点击Open in Terminal，并输入以下指令：
    ```
    ./darknet detector train cfg/cat.data cfg/yolov4-cat.cfg yolov4.conv.137
    ```
* 训练的过程中，生成的权重文件会存放在/darknet/backup文件夹下，训练过程每隔一段时间会生成一个.weights文件。
* 生成.weights文件后，便可以进行测试了（此时训练仍在继续，另外开一个终端进入darknet路径下）。也可以等待全部训练完成后再进行测试。测试命令为：
    ```
    ./darknet detect cfg/yolov4-cat.cfg backup/yolov4-cat_final.weights data/cat_10.jpg
    ```
* 训练时我们可以再开一个终端，通过nvidia-smi命令来查看cuda和显存的使用情况。


但是在训练过程中，我碰到了如下问题：

```
[yolo] params: iou loss: ciou (4), iou_norm: 0.07, obj_norm: 1.00, cls_norm: 1.00, delta_norm: 1.00, scale_x_y: 1.05
nms_kind: greedynms (1), beta = 0.600000 
Total BFLOPS 127.512 
avg_outputs = 1051267 
Loading weights from yolov4.conv.137...
 seen 64, trained: 0 K-images (0 Kilo-batches_64) 
Done! Loaded 137 layers from weights-file 
Learning Rate: 0.001, Momentum: 0.949, Decay: 0.0005
 Detection layer: 139 - type = 28 
 Detection layer: 150 - type = 28 
 Detection layer: 161 - type = 28 
Resizing, random_coef = 1.40 

 896 x 896 
 Create 64 permanent cpu-threads 

 mosaic=1 - compile Darknet with OpenCV for using mosaic=1 

 mosaic=1 - compile Darknet with OpenCV for using mosaic=1 
```

从日志来说感觉看不出来是错误，等于程序终止，我猜想很可能是因为我在Makefile文件里面的opencv=0没有改成opencv=1的原因，但是，要解决这个问题，不仅仅是将0改成1这么简单的操作，还需要配置opencv，而且我在服务器上跑模型，也不需要图形界面，所以我先在Makefile里面查找有没有mosaic=1这行，很可惜，没有，然后我在训练要用的obj.cfg这个文件里面查找，果然，有这一行，如下:

```
#cutmix=1
mosaic=1

#:104x104 54:52x52 85:26x26 104:13x13 for 416

[convolutional]
batch_normalize=1
filters=32
size=3
stride=1
pad=1
activation=mish
```

然后就将这里得1改成0了再去训练模型，没问题了，正常训练。
