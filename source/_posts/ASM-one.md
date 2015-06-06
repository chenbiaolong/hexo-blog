title: ASM(active shape model)算法简介(一)
date: 2015-05-29 16:39:24
tags: ASM
category: 机器学习
mathjax: true
---
##概要
由于hexo对latex支持的不是很好，本文的数学公式显示有些问题，待解决。请访问[这个链接](https://zybuluo.com/chenbiaolong/note/104505)阅读


ASM是一种基于点分布模型（Point Distribution Model,PDM）的算法。在PDM中，外形相似的物体，例如人脸、人手、心脏、肺部等的几何形状可以通过若干关键特征点（landmarks）的坐标依次串联形成一个形状向量来表示。ASM算法需要通过人工标定的方法先标定训练集，经过训练获得形状模型，再通过关键点的匹配实现特定物体的匹配。ASM 的优点是
能根据训练数据对于参数的调节加以限制，从而将形状的改变限制在一个合理的范围内。本文将根据ASM的原始论文和一些资料，整理ASM算法的数学原理。

<!-- more -->
##训练图像的标定
为了建立ASM，需要一组标有n个特征点的N幅人脸图象(包括多个人的不同表情和姿态)作为训练数据。特征点可以标记在脸的外部轮廓和器官的边缘，如下图所示。


![标定的人脸图像](http://7u2qr4.com1.z0.glb.clouddn.com/blog1353147003_8880.png)


这张图中有67个标定点，需要注意的是各个标定点的顺序在训练集中的各张照片需要一致。比如`2`和`12`这两点分别对应脸和耳朵的连接处，在其他的训练图像中也要有一样的标定点。
假设我们一共有`N`张的训练图，每一张图都有`n`个点，第`i`张图像的第`k`点坐标表示为
$$\left(x_{i,j},y_{i,j}\right)$$
对于第i张图像，各个标定点可以用一个矩阵表示：
$$X_i=\left [ x_{i0},y_{i0},x_{i1},y_{i1},...x_{i(n-1)},y_{i(n-1)}\right ]^T$$
其中1<=`i`<=N;
##训练图像的对齐
为了研究训练图象的形状变化，比较不同形状中相对应的点，应先对这些图象进行对齐。对齐是指以某个形状为基准，对其它形状进行旋转，缩放和平移使其尽可能的与基准形状接近的过程。
**与基准形状尽可能接近**，在数学上我们经常用欧式距离的大小衡量接近的程度。假设图像i的标定点矩阵为**Xi**,图像j的标定点矩阵为**Xj**,二者的欧式距离大小为
$$d_{ik}=\sqrt{(x_{i0}-x_{k0})^2+(y_{i0}-y_{k0})^2+(x_{i1}-x_{k1})^2+(y_{i1}-y_{k1})^2+...+(x_{i(n-1)}-x_{k(n-1)})^2+(y_{i(n-1)}-y_{k(n-1)})^2}$$
也可以用如下的矩阵运算形式表示：
$$
d_{ik}^2=(X_i-X_k)^T(X_i-X_k)
$$
有时我们需要对各个点加上不同的权值，设加权矩阵为**W**，加权的欧式距离为
$$
d_{ik}=\sqrt{w_0(x_{i0}-x_{k0})^2+w_0(y_{i0}-y_{k0})^2+w_1(x_{i1}-x_{k1})^2+w_1(y_{i1}-y_{k1})^2+...+w_{n-1}(x_{i(n-1)}-x_{k(n-1)})^2+w_{n-1}(y_{i(n-1)}-y_{k(n-1)})^2}
$$
或者用矩阵运算方式表示:
$$
d_{ik}^2=(X_i-X_k)^TW(X_i-X_k)
$$
其中**W**是对角矩阵
$$
W=diag(w_0,w_0,w_1,w_1,...w_j,w_j,...w_{n-1},w_{n-1})
$$
为了实现训练图像的对齐，我们首先选择一幅基准图像，一般选择第一张图像**X1**。训练集中其他图片经过变换尽可能的接近该基准图像，具体的变化过程可以用一个缩放幅度参数**s**，旋转参数**$\theta$**以及平移参数矩阵**t**表示。假设我们要将第2幅图像经过变换尽可能的接近第1幅图像，变换后的图像矩阵表示为：
$$
M(s,\theta)[X_2]-t
$$
变换后的图像矩阵与基准矩阵**X1**的加权欧式距离平方表示为：
$$
E=（X_1-M(s,\theta)[X_2]-t）^TW（X_1-M(s,\theta)[X_2]-t）
$$
其中
$$
M(s,\theta)\begin{bmatrix} x_{jk}\\y_{jk} \end{bmatrix}=\begin{pmatrix}(s*cos\theta)x_{jk}-(s*sin\theta)y_{jk}\\ (s*sin\theta)x_{jk}+(s*cos\theta)y_{jk}\end{pmatrix}\\
t=(t_x,t_y,t_x,t_y,...t_x,t_y)^T
$$
假设：
$$
a_x=s*cos\theta\\
a_y=s*sin\theta
$$
为了得到最好的对齐结果，我们必须计算出缩放幅度参数**s**，旋转参数**theta**以及平移参数矩阵**t**使**E**达到最小，利用加权最小二乘法可以获得以下方程：
$$
\frac{\partial E}{\partial a_x}=0\\
\frac{\partial E}{\partial a_y}=0\\
\frac{\partial E}{\partial t_x}=0\\
\frac{\partial E}{\partial t_y}=0
$$
分别将**E**代入上面的式子，可以得到下面的矩阵方程：
$$
\begin{pmatrix}X_2 &-Y_2  & W & 0\\ Y_2 &X_2  & 0 & W\\ Z & 0 & X_2 & Y_2 \\ 0 & Z & -Y_2 & X_2\end{pmatrix}\begin{pmatrix}a_x\\ a_y\\ t_x\\ t_y\end{pmatrix}=\begin{pmatrix}X_1\\ Y_1\\ C_1\\C_2\end{pmatrix}
$$
其中
$$
X_i=\sum_{k=0}^{n-1}W_kx_{ik}\\
Y_i=\sum_{k=0}^{n-1}w_ky_{ik}\\
Z=\sum_{k=0}^{n-1}W_k(x_{2k}^2+y_{2k}^2)\\
W=\sum_{k=0}^{n-1}W_k\\
C_1=\sum_{k=0}^{n-1}W_k(x_{1k}x_{2k}+y_{1k}y_{2k})\\
C_2=\sum_{k=0}^{n-1}W_k(y_{1k}x_{2k}+x_{1k}y_{2k})
$$
通过以上方程就可以求出$$a_x,a_y,t_x,t_y$$这4个变量的值，对应的也就能将图像2对齐到图像1了。
在上面的方程中我们还没有讨论如何确定权重矩阵**W**。在训练集的各个点中，权重越高的点对结果的影响也就越大。那么如何确定一个点应该拥有多大的权重呢？在ASM的原始论文中作者选择了趋向比较**稳定**的点。一个稳定的点就是在训练集中与其他点的距离变化不大的点。
假设**R<sub>kl</sub>**代表一张图片中**k**点和**l**点的距离，**V<sub>R<sub>kl</sub></sub>**表示这个距离在所有训练图像中的变化量（原文是let **V<sub>R<sub>kl</sub></sub>** be the variance in this distance over the set of shapes）。在原始论文中并没给出**V<sub>R<sub>kl</sub></sub>**的计算表达式，但按照它的定义应该是可以表示为各张训练图像中**R<sub>kl</sub>**组成的数组的方差形式。

**V<sub>R<sub>kl</sub></sub>**表示在训练集中**k**点和**l**点距离的变化程度，如果将**k**点与其他所有点的距离变化程度进行累加，就可以判断出k点的**稳定**程度。对于权重**W**的确定有如下式子：
$$
w_k=（\sum_{l=0}^{n-1}V_{R_{kl}}）^{-1}
$$
其中n为模型点的个数。
可以看出k点越稳定**，w<sub>k</sub>**越大，**k**点也就有越大的权重。


综合前面的分析，我们可以将训练集中的第2，3，...，（N-1）张图片分别与第一张图像进行对齐，对齐完成后就可以进行下一步的图像分析了。

##图像的PCA分析
前面已经将训练集中的所有图像进行对齐了，接下来将根据对齐后的图像数据进行PCA分析，最终得到训练好的形状模型。

假设我们已经有了N张已经对齐好的图像数据，第i张图像的数据表示为**X<sub>i</sub>**,计算其平均形状为
$$
\bar{X}=\frac{1}{N}\sum_{i=1}^{N} X_i
$$
由于图像的采样点可能比较多，如果直接全部计算所需耗费的计算时间比较大。因此可以利用PCA算法对图像数据进行降维处理。关于PCA的基础理论可以参考[这篇文章][2]。

> 将一组N维向量降为K维（K大于0，小于N），其目标是选择K个单位（模为1）正交基，使得原始数据变换到这组基上后，各字段两两间协方差为0，而字段的方差则尽可能大（在正交的约束下，取最大的K个方差）。

这是原始的定义，但实际上我们对一个形状各个点的原始坐标位置不感兴趣，我们感兴趣的是训练集中各张图像的各个形状点偏离平均形状的大小。因此，在这里我们映射到新空间的不是原始点坐标，而是各个点偏离平均形状的偏离坐标。偏离坐标的协方差矩阵为：
$$
S=\frac{1}{N}\sum_{i=1}^{N}dX_idX_i^T
$$
其中dX<sub>i</sub>为各张图片的形状与平均形状的偏差。
$$
dX_i=X_i-\bar{X}
$$
假设新空间的正交基表示为**P**：
$$
P=\begin{pmatrix}
p_0\\ 
p_1\\ 
...\\
p_{2k-1}\\ 
\end{pmatrix}
$$
根据PCA的相关知识，**P**为**S**的特征向量组成的矩阵,有
$$
Sp_k=\lambda_k p_k
$$

当映射到新的空间后，各个形状偏离平均形状的向量可以用新空间的特征向量线性表达。
$$
dX_i=b_{i0}p_0+b_{i1}p_1+...+b_{i(2n-1)}p_{2n-1}
$$
**b<sub>il</sub>**是第i张图**p<sub>l</sub>**向量的幅度值。至此我们有以下公式
$$
X_i=\bar{X}+dX_i\\
=\bar{X}+Pb_i
$$
根据以上公式可以得到
$$
b_i=P^{-1}(X_i-\bar{X})=P^T(X_i-\bar{X})
$$
这里的b<sub>i</sub>表示第i张图片形状与平均形状的偏离程度。对于一种形状，它在不同图片的表现形式可以用如下公式表示:
$$
X=\bar{X}+Pb
$$
这里**X**表示形状允许的表现形式。比如我们做人脸识别，虽然各个人的长相不一样，并且拍照的角度不一样，但无论如何人脸的形状都是在平均脸的基础上进行一些形变得到的。我们可以规定向量**b**在一个区间范围之内，只要**b**不脱离这个区间，我们就能认为检测到一张人脸。这也是ASM做形状匹配的基础原理。至于**b**的区间范围，在ASM的原始论文中取
$$
-3\sqrt{\lambda_k }\leqslant b_k\leqslant 3\sqrt{\lambda_k }
$$
现在我们已经分析了ASM的图像对齐以及图像形状的PCA分析，在下一篇中我们将介绍如何进行形状的搜索。


##参考文献

[active shape models their training and application(原始论文)](http://personalpages.manchester.ac.uk/staff/timothy.f.cootes/papers/cootes_cviu95.pdf)    


[An Introduction to Active Shape Models]( http://www.ee.oulu.fi/research/imag/courses/Kokkinos/asm_overview.pdf)    


[ASM（Active Shape Model）算法介绍](http://blog.csdn.net/carson2005/article/details/8194317) 


  [1]: http://7u2qr4.com1.z0.glb.clouddn.com/blog1353147003_8880.png
  [2]: http://blog.codinglabs.org/articles/pca-tutorial.html
  [3]: http://personalpages.manchester.ac.uk/staff/timothy.f.cootes/papers/cootes_cviu95.pdf
  [4]: http://www.ee.oulu.fi/research/imag/courses/Kokkinos/asm_overview.pdf
  [5]: http://blog.csdn.net/carson2005/article/details/8194317