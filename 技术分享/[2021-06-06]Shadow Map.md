# Shadow Map

参考资料：GAMES202课程第3讲、Real-Time Rendering第4版第7.4 Shadow Maps

[TOC]

## What is shadow map

一张图片如果没有阴影，显然会显得很假。如何在我们生成的图片里看到阴影，这也是一个很值得探讨的问题。

![07](https://gitee.com/texwood/my-pictures/raw/master/07.png)

虽然光线追踪也可以产生阴影，但是光线追踪所需的计算量十分巨大。如果是在游戏中，巨大计算量带来的时间上的消耗，显然是我们不想要的。因此我们使用shadow map来生成阴影。

shadow mapping是一个两个pass（一个pass表示进行一次顶点以及像素的处理）的算法：第一个pass是light pass，它负责生成SM；第二个pass是camera pass，它使用我们上一个pass生成的SM。



![04](https://gitee.com/texwood/my-pictures/raw/master/04.png)

上面那个式子是我们使用的渲染方程。绿色框框住的是我们生成图像像素的最终颜色，蓝色框是我们在不考虑阴影情况下，这个像素应该渲染出的颜色，而红色的框则是我们的阴影方法。

## Shadow Map

正如我们上一节所说，SM的第一个pass是light pass。

我们从灯光看过去场景，记录离灯光最近的深度。

![05](https://gitee.com/texwood/my-pictures/raw/master/05.png)

如上图，红色的箭头是从灯光出发的一根光线，在这根光线上，蓝色框的物体是可以被灯光照到的，橙色框的物体并不能被灯光照到——橙色框的物体应该有个阴影。

![06](https://gitee.com/texwood/my-pictures/raw/master/06.png)

SM的效果看上去不错。然而现实并没有那么美好。SM有两个大问题：自遮挡和锯齿。

* 自遮挡：如下图，人物的阴影看上去很美好，但是地板上却产生了一圈圈不和谐的“纹路”。本来应该没有阴影的地方却产生了阴影，这是由数值精度造成的现象。SM记录的深度是一个个离散的量，当你camera pass里查询对应位置的深度的时候，可能记录的深度比实际的深度要浅，这就导致了本来可以看见的变成了不可以看见的（这里借用一下闫神手绘的图^^）。

  ![08](https://gitee.com/texwood/my-pictures/raw/master/08.png)

  ![09](https://gitee.com/texwood/my-pictures/raw/master/09.png)

  * 当然自遮挡并不是不能解决的，我们通常使用加一个偏移量来解决自遮挡的问题——也就是我们允许记录值比实际值要小那么一个范围，在这个范围内我们仍然当它是可见的。但这个方法又引入了一个新的问题：那个允许的范围会导致原本就是看不见的物体被看见了。

    ![10](https://gitee.com/texwood/my-pictures/raw/master/10.png)

  * 上面增加一个bias的方法是最常用的方法。除了那种方法以外，我们还可以记录第二近的深度，取第一近和第二近的中间值作为我们需要的深度。找到第一、二近的深度耗时显然比第一近的深度要大，而我们的需求是Real-Time，这样多的耗时不是我们想要的，因此这个方法一般没什么人用。

* 锯齿：我们自然希望每一个SM的纹素能够覆盖图片上的一个像素，然而事实总不能如我们的意思。如果light和camera的位置是相同的，那么SM完美地与屏幕空间地像素一一对应了。然而一旦光的方向改变了，这种与像素对应地比率也会随之改变，这种比率的改变可能会导致如下图所示的现象：

![RTR P240图7.15](https://gitee.com/texwood/my-pictures/raw/master/01.png)

很显然，这个阴影显得一块块的并且十分糟糕，因为SM上一个纹素对应了太多像素。我们通常使用CSM(cascaded shadow maps)来解决这种锯齿，同时它也是Unity内置的方向光实时阴影技术。

## CSM

CSM(*cascaded shadow maps*)，正如它的名字，CSM由多张SM构成。我们将相机的视锥体分割成若干部分（假设数量为N），然后为分割的每一部分生成独立的SM，这些独立的SM通常拥有不同的分辨率，离我们越近的SM分辨率越高。通俗点理解就是，离我们越近我们看到的阴影就越精细，离我们远的物体阴影没那么精细其实也没关系。通常N为1-4中的整数。
* 虽然各个资料上都说是不同的分辨率，但个人认为不同的采样率描述可能会更加准确一些。因为实际操作的时候每张SM的大小通常是一样的，例如：4张1024×1024的SM。这里的分辨率我个人的理解是：虽然是同样大小的SM，但是由于视锥体被分为了几块，每一块相对于light来说，我们都有个不同的透视投影矩阵。就像我们认知的那样，透视投影将物体压缩到某个立方体内。因为视锥体被分为了几块，所以我们能够更紧凑地压缩物体。因此，在同样大小的SM里，更近的物体获得的采样率就会更高，也就是具有更高的分辨率。

![13](https://gitee.com/texwood/my-pictures/raw/master/13.png)

如何划分视锥体是一个十分值得探讨的问题。

$$r=\sqrt[c]{\frac{f}{n}}$$

上面那个公式是RTR4中给出的一种划分方法。利用这个公式我们可以得到每一个划分部分的步长。其中n和f分别为近、远平面距离，c则是我们的划分数量。

由上面的式子，我们不难发现，近、远平面距离是很重要的参数。试想一下，如果近平面离我们太远，那么我们视锥体被划分的那几个部分可能会包含同一个物体，这会导致极大的失真。对于一个固定的场景，我们固然可以人为地去划分，但是我们的游戏显然不是一个固定的场景。

对于实时的场景，有人提出了SDSM(sample distribution shadow maps)，它利用上一帧的Z深度值从两种方法中选取一种：

* 方法一：利用上一帧的最大Z深度和最小Z深度来决定近远平面。
* 方法二：同样分析深度值，并且生成Z深度的直方图，根据直方图我们可以找到更加紧密的近远平面。

实际使用中，方法一更通用，并且更快，也能得到很好的效果，如图（左图是没有使用SDSM，右图则使用了SDSM）：

![03](https://gitee.com/texwood/my-pictures/raw/master/03.png)

## PCF and PCSS

实际生活中，除了上述的硬阴影，更常见的则是软阴影（因为我们日常生活中灯光通常是面光源）。

![14](https://gitee.com/texwood/my-pictures/raw/master/14.png)

* PCF(Percentage Closer Filtering)：如上图所见，软阴影相比硬阴影，它并没有一个明显的界线，或许我们可以理解为：硬阴影的可见性非0即1，软阴影则允许可见性是0.5。如同我们不能在已经生成的图片上做抗锯齿，我们也不会在已经生成的图片上做这样一个模糊操作，同样在SM上做模糊也是毫无意义的——我们在判断某点是否可见的时候做这样一个filtering。我们取该点“周围”点的非0即1的可见性，为这个可见性做一个加权平均，得到了我们的可见程度。如何采样其实也有讲究。如果做过GAMES202作业1，就会知道泊松分布的采样渲染出的效果要好过均匀采样。同时，PCF的size也决定了阴影的软硬程度：size越大越软。
* PCSS(Percentage Closer Soft Shadows)：仔细观察生活中的阴影，我们会发现，离物体越近的阴影越硬，越远则越软。PCSS利用了上述的PCF，并且在PCSS中PCF的大小是自适应的。

![15](https://gitee.com/texwood/my-pictures/raw/master/15.png)

如图，利用三角形的相似，我们能够轻易得到PCF的size：$(\frac{d_{Receiver}}{d_{Blocker}}-1)\cdot w_{Light}$。当然，我们需要的遮挡物深度也得是周围遮挡物的平均深度。
