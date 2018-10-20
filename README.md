Fast-RCNN详解
=============

# 一. 背景

> Fast R-CNN 是 RBG大神基于R-CNN和SPPnet的优化，提出了RoI层代替SPP层，实现了整个物体检测模型大部分网络的end-to-end，进一步提升了速度和准确率。Fast R-CNN 采用非常深的检测网络（例如VGG16）训练，速度比R-CNN快9倍，比SPPnet快3倍。在运行时，检测网络在PASCAL VOC 2012数据集上实现最高准确度，其中mAP为66％（R-CNN为62％），每张图像处理时间为0.3秒，不包括候选框的生成。测试时间比R-CNN快213倍。

# 二. R-CNN和SPPnet的缺陷

## (一) R-CNN的缺陷

### 1. 训练过程是多级流水线(multi-stage pipeline)

> R-CNN首先使用目标候选框对卷积神经网络使用log损失进行微调。
>
> 然后将卷积神经网络得到的特征送入SVM，这些SVM作为目标检测器，替代通过微调学习的softmax分类器。
>
> 在第三个训练阶段，SVM学习检测框回归。

### 2. 训练时时间和空间开销大


> 对于SVM和检测框回归训练，需要从每个图像中的每个目标候选框提取特征，并写入磁盘。对于非常深的网络，如VGG16，这个过程在单个GPU上需要2.5天（VOC07 trainval上的5k个图像），这些特征需要数百GB的存储空间。

### 3. 目标检测速度慢

> 在测试时，需要从每个测试图像中的每个目标候选框提取特征，用VGG16网络检测目标每个图像需要47秒（在GPU上）。
>

#### R-CNN很慢是因为它为每个目标候选框进行卷积神经网络正向传递，而不共享计算。空间金字塔池化网络（SPPnets）则通过共享计算加速了R-CNN。

## (二) SPPnet的缺陷

> 分阶段训练网络：选取候选区域、训练CNN、训练SVM、训练bbox回归器
>
> 特征需要写入磁盘
>
> 训练SVM，bbox回归时算法不能更新卷积层的参数，这会影响网络的精度

# 三. Fast-RCNN

> Fast-RCNN设计思想如图所示：

![image](https://github.com/ShaoQiBNU/Fast_RCNN/blob/master/images/1)

![image](https://github.com/ShaoQiBNU/Fast_RCNN/blob/master/images/2.png)

```
1. 任意size图片输入CNN网络，得到feature map

2. 在任意size图片上采用selective search算法提取约2k个建议框；

3. 根据原图中建议框到feature map映射关系，在feature map中找到每个建议框对应的区域即ROI；

4. 利用Spatial pooling的思想，对每个ROI做ROI pooling得到HxW（例如：VGG-16网络是7×7）的特征图，然后得到特征向量；

5. 将该特征向量feed进全连接层得到固定大小的特征向量；

6. 5所得特征向量经由各自的全连接层——由SVD分解实现，分别得到两个输出向量：一个是softmax的分类得分，一个是Bounding-box窗口回归；

7. 利用窗口得分分别对每一类物体进行非极大值抑制剔除重叠建议框，最终得到每个类别中回归修正后的得分最高的窗口。
```

## (一) ROI pooling layer

> ROI pooling layer使用最大池化将任何有效区域内的特征转化成一个小的带有固定空间范围HxW（比如7×7）的特征图，其中H和W是层的超参数，和任何特定的ROI无关。本文中，一个ROI是针对卷积特征图的一个矩形窗口。每个ROI定义成四元组（r, c, h, w），左上角为（r, c），高和宽是（h, w）。
>
> ROI最大池化将hxw的ROI窗口分成HxW的子窗口网格，每个子窗口大小大约是h/H x w/W。然后每个子窗口进行最大池化放入网格对应的单元，池化以标准最大池化的形式独立应用在每个特征图的channel上。ROI层是SPPnets中的空间金字塔层的一个特例，因为是一个一层的金字塔结构。

## (二) 预训练网络初始化

>  首先对于pretrained ImageNet CNN，把最后一个pooling layer改为ROI pooling layer，并且设置H和W使得spatial pooling feature的维度和全连接层第一层的结点数一致。把网络的最后一个全连接层和softmax层换成两个输出层（object分类和bbox回归）。本文采用VGG16网络，将该网络最后一个pool层替换为ROI pooling layer，由于VGG16最后一个pool层出来的大小为7x7，所以ROI pooling layer需要将所有的ROI池化成7x7的特征图，然后feed进全连接层。

![image](https://github.com/ShaoQiBNU/Fast_RCNN/blob/master/images/3.png)

