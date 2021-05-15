# Ambient Occlusion #

什么是环境光遮蔽？：物体被环境光照（太阳等）的遮蔽情况

In Blinn-Phong:近似为常数值，非常不真实

光栅化计算的局限：遮蔽关系往往是两个物体间的，光栅化只有单个物体的信息

使用光线追踪或者后处理

## 基本思想 ##

 从某一点开始以半球形状发射线段，步进判断是否被物体遮蔽

缺点很明显，射线不够多，效果很烂；射线太多，不能用于实时渲染

## SSAO ##

SS：ScreenSpace

* 只计算自己能看见的（毕竟看不到的计算了也没用），也就是后处理
* 计算是否被遮蔽时，将步进点投射到屏幕中，比较深度即可

缺点：深度精确度；采样数；糟糕的随机情况

UE中的改进：

* TAA 抖动 历史值混合
* 随机方向的不同策略（用固定方向的伪随机代替或者部分代替随机射线的生成）
* 半分辨率混合（一半屏幕分辨率，再次计算然后混合，比模糊效果好）
* 法线重建（根据深度贴图计算更可信的法线）

## DFAO ##

DF：(Signed)DistanceField有向距离场

* 几何物体的表示
  * 显式：明确指定点的坐标
  * 隐式，使用距离关系等几何约束表示，如x^2 + y^ 2 =r ^2

有向距离场就是隐式表示的一种，用特定点与平面之间的距离（和朝向）来表示几何物体

优点：保持几何特性（如圆，可能远处看着还行，近处无法解决）；便于射线求交运算等

缺点：不够精确（就算是算距离也是比较模糊的），光栅化用别想了



SDF计算：

* UE里：一个个Grid，从中心点发出射线，根据距离和射线方向计算结果(MeshDistanceFieldUtilities.cpp)
* 其他：KD树，BVH等（可以看Continuous signed distance field representation of polygon mesh）



但给AO使用，最好像SSAO一样，只取用自己需要的就好

Object SDF->Global Distance Field 更新距离字段就行（还有根据个部分情况决定是否更新等优化）

计算AO同基础思路



相较于其他，DFAO需要提前设置与计算，精度有限，但计算速度快，适用于动态物体

###距离场的其他应用 ###

高精度图像，妈妈再也不用担心放大字体的锯齿（Unity的Text Mesh）

软阴影：快但精度不行，近处还是得用级联阴影（精度正义执行）

##其他计算方式 ##

### HBAO ###

还是屏幕里的一个点，但是射线的步进发生在屏幕空间中，直接看深度值

然后会拿到一个最大角度（height / distance），记作a

根据法线，拿到面与这个水平场的角度b(插值得到的法线可能不太可行，应该使用面法线)

sin a - sin b就完了

其他优化：

* 还是分辨率不行，模糊或者半分辨率来搞
* 高度变化过大，不能准确表示（乘以一个与distance有关的小权值完事）

==GTAO==：个人了解的不多，基本上是来自于HBAO的优化，水平场变成圆切面，几个圆弧切入



一点资料：

[距离场环境光遮蔽 ](https://docs.unrealengine.com/zh-CN/BuildingWorlds/LightingAndShadows/DistanceFieldAmbientOcclusion/index.html)

[距离场柔和阴影 ](https://docs.unrealengine.com/zh-CN/BuildingWorlds/LightingAndShadows/RayTracedDistanceFieldShadowing/index.html)

[TAA](https://zhuanlan.zhihu.com/p/366494818)[抗锯齿](https://zhuanlan.zhihu.com/p/366494818)

[HBAO](https://zhuanlan.zhihu.com/p/103683536)[(](https://zhuanlan.zhihu.com/p/103683536)[屏幕空间的环境光遮蔽](https://zhuanlan.zhihu.com/p/103683536)[)](https://zhuanlan.zhihu.com/p/103683536)

http://advances.realtimerendering.com/s2015/DynamicOcclusionWithSignedDistanceFields.pdf