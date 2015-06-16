title: ASM(active shape model)算法简介(二)
date: 2015-06-16 17:17:20
tags: ASM
category: 机器学习
mathjax: true
---
##概要
在上一篇文章中，我们利用PCA分析根据训练集得到了形状的平均模型。一个形状可以表示为$X=\bar{X}+Pb$，$\bar{X}$为平均形状，从公式可以看出选定不同的$b$可以得到形状的不同形态。我们将$b$称为形状参数(shape parameter)。举个例子来说，在人脸识别中，$\bar{X}$是训练集中各个人的平均脸，表示“完美”的人脸。而$Pb$表示实际图片中的人脸偏离“完美人脸”的程度。在这篇文章中我们将讨论在一张图片中ASM算法是如何逐步找到特定形状的。
<!-- more -->

##ASM算法图像搜索过程

###为每一个特征点构建局部特性

因为给定的平均形状未必与当前图形的形状一致，所以需要对平均形状进行调整并使其与真正的形状最接近，即找到每个点的最佳位置。为此，首先分析训练集中每幅图象的局部灰度信息。对每个形状的每一个特征点，在经过此点且沿与其邻近点垂直方向上取$2m+1$个象素的灰度并对其标准化作为局部灰度信息,称为一个profile。局部灰度信息特征的创建过程如下：

![此处输入图片的描述](http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150615170812.png)

如上图所示，在第i个训练图像的第j个特征点两侧，沿着垂直于该点前后两个特征点连线的方向上分别选择m个像素以构成一个长度为2m+1的向量，这2m+1维的向量叫做特征点j的profile，我们用$g_{ij}$表示第i张图第j个特征点的profile。
$$
g_{ij}=(g_{ij0},g_{ij1},g_{ij2},...,g_{ij2m})^T
$$
其中$g_{ijk}$表示$2m+1$个像素点中第k个像素点的灰度值。我们再对特征点的profile进行一些处理，将相邻的像素点的灰度信息相减(即数字图像的求导)，得到$2m$维度的$dg_{ij}$,称为**derivative profile**
$$
dg_{ij}=(g_{ij1}-g_{ij0},g_{ij2}-g_{ij1},...,g_{ijn_{p-1}}-g_{ij(2m-1)})^T
$$

接下来我们再求各个训练图像中特征点j的**derivative profile**平均值
$$
\bar{g_j}=\frac{1}{N}\sum_{i=1}^{N}dg_{ij}
$$
以及方差矩阵
$$
C_{j}=\frac{1}{N}\sum_{i=1}^{N}(g_{ij}-\bar{g_j})(g_{ij}-\bar{g_j})^T
$$
至此，我们得到了形状特征点j的局部灰度模型。这个灰度模型包括的信息是在特征点附近的灰度梯度信息和灰度值方差信息。在新的图像中我们只需将一个位置的灰度信息和我们得到的上述模型的局部灰度信息进行比较，相似度越高则说明位置越好。匹配点从初始位置出发，逐渐逼近最优位置。特征点j的新特征$s_j$与其训练好的局部特征之间的相似程度可以用马氏距离表示：
$$
f_{sim}=(s_j-\bar{g_j})C_{j}^{-1}(s_j-\bar{g_j})^T
$$
其中$s_j$的计算过程与$\bar{g_j}$类似

###计算每个特征点的新位置

在上一小节中,我们根据训练集获得了特征点$j$的局部灰度信息模型$\bar{g_j}$,并且得到了与该局部灰度信息模型相似度的计算公式：
$$
f_{sim}=(s_j-\bar{g_j})C_{j}^{-1}(s_j-\bar{g_j})^T
$$
我们将根据这2个信息计算在搜索图像（非训练图像）中寻找各个特征点的新位置。首先先把初始ASM模型覆盖在搜索图像上，模型上各个特征点都有一个初始位置。对于第$j$个特征点，在垂直于前后两个特征点连线上以其为中心两边各选择$l$（$l$>$m$）个像素,然后计算这$l$个像素的灰度值导数并归一化从而得到一个局部特征，其包括$2(l-m)+1$个子局部特性，然后利用前面的公式计算这些子局部特性与当前特征点的局部特性之间的马氏距离，使得马氏距离最小的那个子局部特性的中心即为当前特征点的新位置，这样就会产生一个位移。示意图如下：

![此处输入图片的描述](http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150615195435.png)

通过这个流程我们可以计算得到各个特征点的新位置，它们相对于初始位置的位移组成一个向量$dX$,
$$
dX=(dX_1,dX_2,...dX_n) 
$$
其中n代表形状的特征点个数。

###模型参数更新

在上一篇文章中我们将形状表示为：
$$
X=\bar{X}+Pb 
$$
其中
$b=(b_1,b_2...,b_k)^T$,
$-3\sqrt{\lambda_i }\leqslant b_i\leqslant 3\sqrt{\lambda_i }; (0<i\leqslant k)$

为了表述方便，我们将这个式子中的$X$表示为基准形状，而$\bar{X}$表示为平均形状。平均形状调整形状参数就可以得到不同的基准形状。同样用人脸作为例子，$\bar{X}$是训练集中人的平均脸，而基准形状$X$可以表示为照片中某个人（假设为张三）的人脸，对基准形状进行旋转、缩放、平移就是张三的不同照片，旋转缩放平移参数称为姿态参数（pose parameter）。

我们先假定一个基准形状的初始姿态$X_i$(张三的一张照片)，它可由一个基准形状$X$（张三的脸）经过旋转$\theta$，缩放$s$和平移得到,表示为：
$$
X_i=M(s,\theta)[X]+X_c
$$
其中$X_c=(x_c,y_c,x_c,y_c,...x_c,y_c)^T$,$(x_c,y_c)$是图像模型的中心。

经过前一小节的分析，我们得到形状的一个位移向量$dX_i$。由于我们搜索的是一种**形状**，而$dX_i$是只是各个点与初始位置的偏移向量，我们无法将$dX_i$与上一篇文章中建立的形状模型相联系。

在这一小节我们将$dX_i$与上一篇文章建立的模型进行联系。由于模型建立好后的$\bar{X}$与特征向量$P$都已经确定，当模型变化时变化的参数只能是$b$参数，假设变化量为$db$。因此我们必须建立起$dX_i$与$db$的映射关系。

为了使$X$经过形状参数变化和姿态参数变化使其与$X_i+dX_i$最接近，我们需要改变$X_i=M(s,\theta)[X]+X_c$式中的$X$为$X+dX$。将其姿态参数中的平移量，旋转角度和缩放比例比例分别改变$(dx_c,dy_c),d\theta$和$(1+ds)$。表示为下式：

$$
M(s(1+ds),\theta+d\theta)[X+dX]+(X_c+dX_c)=X_i+dX_i
$$
由于$X_i=M(s,\theta)[X]+X_c$上面的公式也可以表示为:
$$
M(s(1+ds),\theta+d\theta)[X+dX]=M(s,\theta)[X]+dX_i-dX_c
$$
由于
$$
M^{-1}(s,\theta)[...]=M(s^{-1},-\theta)[...]
$$
可以得到：
$$
X+dX=M((s(1+ds))^{-1},-(\theta+d\theta))[M(s,\theta)[X]+dX_i-dX_c]
$$
为了表述方便，令$y=M(s,\theta)[X]+dX_i-dX_c$,则上式可表示为
$$
X+dX=M((s(1+ds))^{-1},-(\theta+d\theta))[y]
$$
可以求得$dX$的表达式：
$$
dX=M((s(1+ds))^{-1},-(\theta+d\theta))[y]-X
$$
等式里面的姿态参数变化量$(1+ds,d\theta,dX_c)$可以通过上一篇文章中图像对齐的算法求得。

我们希望找到这么一个$db$使得以下的式子成立：
$$
X+dX\approx \bar{X}+P(b+db)
$$
联立$X=\bar{X}+Pb$，求得：
$$
dX \approx P(db)
$$
所以
$$
db=P^TdX
$$
 
##参数更新

我们总结一下参数更新的流程。

 1. 我们有一个初始形状$X$，并且初始形状有一个初始的姿态$X_i$。$X=\bar{X}+Pb,X_i=M(s，\theta)[X]+X_c$
 
 2. 我们根据局部灰度信息，得到一个位移向量$dX_i$,这个位移向量使当前图像的模型点移动到更好的位置。
 
 3. 我们利用上一篇文章中的图像对齐算法调整姿态参数使$X_i$尽可能的接近$X_i+dX_i$，进而获得姿态参数的变化量$(1+ds,d\theta,dX_c)$。
 
 4. 根据上面的姿态参数（pose parameter）变化量，我们获得形状参数（shape parameter）的变化:
 $$db=P^TdX\\
 dX=M((s(1+ds))^{-1},-(\theta+d\theta))[y]-X\\
 y=M(s,\theta)[X]+dX_i-dX_c
 $$
 
 5. 对姿态参数和形状参数进行更新:
 $$
 X_c-->X_c+dX_c
 $$
 $$
 s-->s+ds
 $$
 $$
 \theta-->\theta+d\theta\\
 $$
 $$
 b-->b+db
 $$
 
 
 6. 参数更新后我们得到一个新的形状$X^{(1)}$,以及该形状的姿态$X_i^{(1)}$,并且有
$$
X^{(1)}=\bar{X}+Pb^{(1)},X_i^{(1)}=M(s^{(1)}，\theta^{(1)})[X^{(1)}]+X_c^{(1)}
$$
其中
 $$
 X_c^{(1)}=X_c+dX_c
 $$
 $$
 s^{(1)}=s+ds
 $$
 $$
 \theta^{(1)}=\theta+d\theta\\
 $$
 $$
 b^{(1)}=b+db
 $$

7.返回1进行迭代计算，直到参数变化不大

##小结
本文对ASM算法进行了总结，由于这两篇文章是我根据ASM论文和一些材料的个人理解写成的，可能有些错误，如果有疑惑，请以原始论文为准。