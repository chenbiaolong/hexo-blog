title: SVD(矩阵奇异值分解)简介
date: 2015-07-01 20:39:52
tags: SVD
category: 机器学习
mathjax: true
---

##概述

线性代数方法使我们可以将数据分解为很多分量，然后分析其中的主要分量，奇异值分解(SVD)就是其中的一种矩阵分解技术，它是一种有效的代数特征提取方法，深刻揭示了矩阵的内部结构。目前，奇异值分解在信息检索方面的应用主要是隐含语义检索(Latent Semantic Indexing，LSI)。潜在语义索引可以将文档在向量空间模型中的高维表示，投影到低维的潜在语义空间中，这一方面缩小了检索的规模，另一方面也在一定程度上避免了数据的过分稀疏。本文主要根据下文列出的参考文献进行整理，并加入了一些个人理解。
<!-- more -->

##基础概念

SVD的具体数学定义这里不详细说明了，简单的说一个m*n的矩阵A可以表示为：
$$
A=U\Sigma V^T
$$
假设A是一个M*N的矩阵，那么得到的U是一个M*M的方阵（里面的向量是正交的，U里面的向量称为左奇异向量），Σ是一个M*N的矩阵（除了对角线的元素都是0，对角线上的元素称为奇异值），$V^T$是一个N * N的矩阵，里面的向量也是正交的，V里面的向量称为右奇异向量）。

###SVD的几何意义

我们知道一个矩阵可以看成一个线性变换，因为一个矩阵乘以一个向量后得到的向量，其实就相当于将这个向量进行了线性变换。比如一个二维的对角矩阵：
$$
M=\begin{bmatrix}
3 & 0\\ 
0 & 1
\end{bmatrix}
$$
从几何的角度，平面上的点（x, y）乘以M变换成另外一个点（3x, y）。
这种变换的效果如下：平面在水平方向被拉伸了3倍，在竖直方向无变化。

![此处输入图片的描述]( http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150701155957.png)



对角矩阵的变换是有一个很好的特性就是它是**解耦**的，即它的变换是各个维度相互独立的。如果是一个非对角矩阵，那么它在几何变化方向就不是“正”的x和y方向了。比如如果
$$
M=\begin{bmatrix}
2 & 1\\ 
1 & 2
\end{bmatrix}
$$
它的几何变换结果为:

![此处输入图片的描述](http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150701163654.png)



很难用简单的语言描述上面图形是如何变换的，因为对于不同的点它的变化方向是不一样的。（x,y）经过变换变为(2x+y,x+2y),点在x轴的变化量需要考虑点在y轴上的坐标，同理点在y轴的变化也要考虑x轴的坐标。这种变换x和y方向是相互**耦合**的，而在实际问题分析中我们并不希望这种耦合。我们将上面的图片旋转45度，得到以下结果：

![此处输入图片的描述]( http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150701165534.png)


我们发现，现在变换后的图像在x轴和y轴的变化已经解耦了。

当A是方阵时,其奇异值的几何意义是:若x是n维单位球面上的一点,则Ax是一个n维椭球面上的点,其中椭球的n个半轴长正好是A的n个奇异值。简单地说,在二维情况下,A将单位圆变成了椭圆,A的两个奇异值是椭圆的长半轴和短半轴，如下图（图片来自维基百科）：

![此处输入图片的描述](http://7u2qr4.com1.z0.glb.clouddn.com/blog512px-Singular-Value-Decomposition.svg.png)


左上表示原来的图像，右上表示经过M矩阵变换后的最后图像。这个变换可以分解为3步：首先经过V*矩阵旋转坐标系，然后通过$\Sigma$对角矩阵在新的坐标系上进行拉伸（新的坐标系下拉伸方向是解耦的），最后再通过U矩阵再进行坐标系旋转，得到最终结果。维基百科对这张图的原文描述如下：

> The upper left shows the unit disc in blue together with the two canonical unit vectors. The upper right shows the action of M on the unit disc: it distorts the circle to an ellipse. The SVD decomposes M into three simple transformations: a rotation V*, a scaling Σ along the coordinate axes and a second rotation U. The SVD reveals the lengths σ1 resp. σ2 of the of the semi-major axis resp. semi-minor axis of the ellispe; they are just the singular values which occur as diagonal elements of the scaling Σ. The rotation of the ellipse with respect to the coordinate axes is given by U.

SVD的意义我觉得可以这么理解：它可以通过对原坐标系进行处理，使得原先相互耦合的变换转换成相互独立维度变换。

##SVD的应用

本小节主要介绍SVD在LSI的应用。潜在语义索引(LSI)，又称为潜在语义分析(LSA)，是在信息检索领域提出来的一个概念。主要是在解决两类问题，一类是一词多义，如“bank”一词，可以指银行，也可以指河岸；另一类是一义多词，即同义词问题，如“car”和“automobile”具有相同的含义，如果在检索的过程中，在计算这两类问题的相似性时，依靠余弦相似性的方法将不能很好的处理这样的问题。所以提出了潜在语义索引的方法，利用SVD降维的方法将词项和文本映射到一个新的空间。我们用下面的例子进行说明：

![此处输入图片的描述]( http://7u2qr4.com1.z0.glb.clouddn.com/blog201101192226386634.png)



 这就是一个矩阵，不过不太一样的是，这里的一行表示一个词在哪些title中出现了，一列表示一个title中有哪些词。比如说T1这个title中就有guide、investing、market、stock四个词，各出现了一次。我们将这个矩阵进行SVD，由于M矩阵是11X9的矩阵，按照SVD分解应该得到一个11X9的对角矩阵，即应该有9个奇异值。
 这9个奇异值的能量大小分布如下图：
 
 ![此处输入图片的描述]( http://7u2qr4.com1.z0.glb.clouddn.com/bloghist.png)


我们发现，大部分能量都集中在前几个奇异值上。这里我们取前3个值得到下面的矩阵：

![此处输入图片的描述]( http://7u2qr4.com1.z0.glb.clouddn.com/blog201101192226397148.png)



左奇异向量表示词（矩阵的行）的一些特性，右奇异向量表示文档（列）的一些特性，中间的奇异值矩阵表示左奇异向量的一行与右奇异向量的一列的重要程序，数字越大越重要。

因此SVD可以挖掘出数据中潜在的“重要因素（factor）”。奇异值越大，该factor对结果的影响就越大。SVD可以利用这个特性做用户推荐。反之，如果我们的M矩阵是一个杂乱无章的矩阵，我们对该矩阵进行SVD分解将会得到相差无几的奇异值，也就无法提取出对结果影响比较大的factor了。

##参考文献

http://www.cnblogs.com/leftnoteasy/archive/2011/01/19/svd-and-applications.html
http://blog.csdn.net/google19890102/article/details/29591553
http://www.puffinwarellc.com/index.php/news-and-articles/articles/33-latent-semantic-analysis-tutorial.html?start=4
http://www.ams.org/samplings/feature-column/fcarc-svd
维基百科