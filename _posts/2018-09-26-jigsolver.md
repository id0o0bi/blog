---
layout: page
title:  "用OpenCV完成拼图的尝试"
date:   2018-09-26 16:30:00 +0800
category: tech opencv
permalink: jigsolver.html
---

### 搁置的项目不要太多

没看完书，没写完的东西，没做完的事情，甚至一些一直想做都没开始的事情。累积得越多，最终就会放弃的越多。要时常提醒自己，做事情善始善终。行必果。不能让放弃成为一种习惯（一堆废话）！

### 思路

从人的角度出发，我们去做拼图的时候，一般是下面几个步骤：
* 先把颜色相近的图片分成小堆
* 从小堆里找出特征明显（有清晰线条）的图片拼合
* 把拼合好的小分块拼成大分块
* 将剩余的图片填充进打分片

我用上面的方法，已经人肉拼完了《日本桥》，接下来睡莲1500片，必需用CV协助解决了。
<!-- ![bridge](/assets/post-images/jigsolver/bridge.jpeg) -->

### 现成的解决方案？

百度一下，没有，问谷歌，不存在！GitHub找一圈，貌似有，打开一看居然是方块图。。。WTF，方块的拼图也能叫jigsaw puzzle吗？

看来只能从我人肉拼图的经验，自己写点东西了。

所谓的经验，其实就是拼图的缺角几乎每个都是不相同的，能够完全互相匹配的，基本上只有对应的那一片。

从这点出发，如果能获取到每个拼图的边缘数据信息，利用对应的边缘匹配，或许能有所收获。所以第一步，先获取每个单片的边缘信息。

### the JigSolver

#### Step 1. 获取单片边缘信息
项目放到MLiA仓库里面了，代码已经改过好几次。基本环境如下：
* OpenCV
* Python
* Numpy

首先：单片的图片是直接用手机拍摄的。默认白色纸背景，方便提取边缘信息。原图如下：
![jigsawpiece](/assets/post-images/jigsolver/jig004.jpeg)

可以看到，由于拼片有一定厚度，在灯光下产生了一部分阴影。这影响了边缘检测的准确度（我可能需要一个拍照的灯箱？）

第一步：我是先把背景手动过滤掉了，由于是接近白色的背景，过滤的方法还是比较好写的。过滤完了之后把白色背景替换成全黑色，增加对比。  
第二步：标准边缘检测处理：高斯滤波，增强，腐蚀，膨胀。最终将边缘线画出来如下图：

![edges01](/assets/post-images/jigsolver/1548526144724.jpg)
![edges02](/assets/post-images/jigsolver/1548526319144.jpg)

可以看出来，深色由于和背景色对比更明显，边缘提取出来的很清晰，含有浅色的拼片线条就不是很规整。不过整体来看，提取出来的信息还是比较可靠的。当然更高的拍摄质量可能更容易提高边缘提取的准确度。

OpenCV提取出来的边缘（contour）是一连串的像素点连成的曲线，这个在后面的线条匹配中需要用到。

Step 1 到这里可以说阶段性的完成了，当然全部的拼片我还没有统一拍照处理，目前只是拿几个测试。

接下来就是 Step2

#### Step 2. 边缘信息匹配

现在面对两个问题：
1. Step 1 提取的边缘是一个闭环曲线，我是否要把四条边单独分出来。
2. 两个不规则的曲线，如何判断是否吻合。   

![edges02](/assets/post-images/jigsolver/problem.jpg)  
第一个问题我想通过一定的数学方法，解决起来应该是没问题的。  
第二个就很头疼。昨天疯狂search，看到有人推荐迭代最近点算法（ICP）。  
简单看了下ICP算法，居然是应用在三维曲面拟合的算法，所以在二维环境里也可以适用，不过目前看到的方法也都是计算方差，貌似和机器学习的梯度下降很相似啊。而且同样存在局部最优解问题。搜到几篇二维应用的文章，还在探索中。

#### Step 3. 多块拼图如何处理


### PS:
发现两篇研究这个课题的论文：  
`Stanford` [Using Computer Vision to Solve Jigsaw Puzzles](https://web.stanford.edu/class/cs231a/prev_projects_2016/computer-vision-solve__1_.pdf)   
`Springer` [An Innovative Algorithm for Solving Jigsaw Puzzles
Using Geometrical and Color Features](https://link.springer.com/content/pdf/10.1007%2F11578079_99.pdf)