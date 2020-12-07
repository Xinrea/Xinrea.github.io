---
layout: post
title: "🥒[计算机图形学]Points和Vectors的变换"
date: 2018-09-29
tags:  Computer-Graphics
---

Points和Vectors均采用1x3的矩阵来表示，Points和Vectors的区别在于，Points具有绝对的位置，因此可以进行平移变换，也可以绕某点进行旋转变换；Vectors指示了方向，平移变换并不改变其方向，因此不存在平移变换。

通过构建合适的矩阵，将1x3矩阵与3x3矩阵相乘，即可得到目标结果，而所构建的用于产生变换的3x3矩阵即为变换矩阵。

$$
\begin{bmatrix}x & y & z\end{bmatrix} *
\begin{bmatrix}
c_{00}&c_{01}&{c_{02}}\\
c_{10}&c_{11}&{c_{12}}\\
c_{20}&c_{21}&{c_{22}}\\
\end{bmatrix}
$$

## 单位矩阵 （The Identity Matrix）

对于下面的这个特殊的矩阵（单位矩阵 - Identity Matrix）来说，运算未产生任何变换。

$$
\begin{bmatrix}
\color{red}{1} & 0 & 0 \\
0 & \color{red}{1} & 0 \\
0 & 0 & \color{red}{1}
\end{bmatrix}
$$

运算如下所示：

$$
    Ptransformed.x = P.x * 1 + P.y * 0 + P.z * 0 = P.x \\
    Ptransformed.y = P.x * 0 + P.y * 1 + P.z * 0 = P.y \\
    Ptransformed.z = P.x * 0 + P.y * 0 + P.z * 1 = P.z 
$$

结果是`Ptransformed = P`。

## 缩放矩阵 （The Scaling Matrix）

从单位矩阵的运算结果中已经可以看出对角线上的系数（coefficients）对结果的影响了，稍作修改，即可得到缩放矩阵。

$$
\begin{bmatrix}
\color{red}{c_{0}} & 0 & 0 \\
0 & \color{red}{c_{1}} & 0 \\
0 & 0 & \color{red}{c_{2}}
\end{bmatrix}
$$

## 旋转矩阵 （The Rotation Matrix）

![旋转矩阵分析](https://www.scratchapixel.com/images/upload/geometry/unitcircle.png?)

通过上图，我们来分析在旋转变换下，x、y、z会发生怎样的变化。假设横轴为x，纵轴为y，也就是说以z为轴进行旋转。

显然有以下结果：

$$
l = \sqrt{x^{2}+y^{2}}\\
x = cos\theta*l\\
y = sin\theta*l\\
$$

则有：

$$
\begin{align}
x'&=cos(\theta+\theta')*l\\
&=l*(cos\theta*cos\theta'-sin\theta*sin\theta')\\
&=x*cos\theta'-y*sin\theta'\\
\end{align}
$$

$$
\begin{align}
y'&=sin(\theta+\theta')*l\\
&=l*(sin\theta*cos\theta'+cos\theta*sin\theta')\\
&=x*sin\theta'+y*cos\theta'\\
\end{align}
$$

这意味着，对于3x3的变换矩阵，只要控制系数c使得相乘结果与上述一致，即完成了旋转操作。对于绕z轴旋转的操作，可由以下变换矩阵实现。

$$
R_{z}(\theta’)=
\begin{bmatrix}
cos\theta' & sin\theta' & 0\\
-sin\theta' & cos\theta' & 0\\
0 & 0 & 1
\end{bmatrix}
$$

同理，还可以得到绕x或y旋转所需的变换矩阵形式：

$$
R_x(\theta')= \begin{bmatrix} 
1 & 0 & 0 \\
0 & \cos(\theta') & \sin(\theta') \\
0 & -\sin(\theta') & \cos(\theta') \\
\end{bmatrix}
$$

$$
R_y(\theta')= \begin{bmatrix}
\cos(\theta') & 0 & -\sin(\theta') \\
0 & 1 &  0 \\
\sin(\theta') & 0 & \cos(\theta') \\ 
\end{bmatrix}
$$

### 所以旋转矩阵中究竟包含了什么信息？

通过目标结果与初始量之间的分析，我们构造出了绕某轴旋转时的变换矩阵；这样的构造使得运算能够得到正确的结果，但是我们还没有弄清楚旋转矩阵中究竟包含了什么信息，使得其能够完成旋转变换。

实际上，矩阵中包含了有关坐标系的信息。旋转矩阵的三行分别代表了x、y、z三个轴旋转\theta'后的结果。可以这样思考：

变换矩阵可以通过矩阵的乘法来完整多种变换的融合，若要实现旋转操作，实际上可以拆分为如下所示：

$$
P*R=(P*I)*R=P*(I*R)
$$

其中，`I*R`即为单位矩阵旋转后所得的矩阵，而单位矩阵I的三行实际上代表了坐标系的x、y、z三轴。

## 旋转平移矩阵

对于Points来说，其平移操作虽然只需要矩阵的加法即可完成，但由于旋转变换使用了矩阵的乘法，为了变换的统一，即能够将旋转和平移操作合并到一个矩阵中，需要对前述的各种设定进行细微的修改，以便在旋转矩阵中加入平移变换。

从变换的结果来看，P经过旋转和平移变换为P'，结果如下：

$$
\begin{array}{l}
P'.x = P.x * M_{00} + P.y * M_{10} + P.z * M_{20} + T_X\\
P'.y = P.x * M_{01} + P.y * M_{11} + P.z * M_{21} + T_Y\\
P'.z = P.x * M_{02} + P.y * M_{12} + P.z * M_{22} + T_Z
\end{array}
$$

与旋转操作对比，只是在旋转操作完成后加上了平移的偏移值。但是为了能够在矩阵乘法过程中完成这个操作，显然3x3的矩阵是无法办到的。

若将矩阵扩展为4x4，为了使Points能够与其相乘，因此也需要将Points的表现形式修正为1x4的矩阵，由于这个修正只是为了引入偏移量，因此默认为1。修改设定后，变换操作的过程可以表现为以下形式：

$$
\begin{array}{l}
P'.x = P.x * M_{00} + P.y * M_{10} + P.z * M_{20} + 1 * M_{30}\\
P'.y = P.x * M_{01} + P.y * M_{11} + P.z * M_{21} + 1 * M_{31}\\
P'.z = P.x * M_{02} + P.y * M_{12} + P.z * M_{22} + 1 * M_{32}\end{array}
$$

显然，变换后的结果应仍为Points，因此单纯地将旋转矩阵添加一行为4x3是不妥的，这也是我们如上述将矩阵直接扩展为4x4的原因。

当然，扩展后，1x4的Points与4x4的变换矩阵相乘，结果中Points最后一位的计算如下所示：

$$
w = P.x * M_{03} + P.y * M_{13} + P.z * M_{23} + 1 * M_{33}
$$

当然，为了使结果得到的w也为1，我们可以选择默认旋转平移矩阵的第4列为0001，这样使得w恒为1；也可以用这个第4列来实现新的变换，即在得到w的值后，将Points矩阵缩放使得w为1，这样使得第四列可以用于缩放变换。

经过上述讨论后，可以得到绕z轴旋转的旋转平移缩放矩阵的形式：

$$
R_{z}(\theta’)=
\begin{bmatrix}
cos\theta' & sin\theta' & 0 & 0\\
-sin\theta' & cos\theta' & 0 & 0\\
0 & 0 & 1 & 0\\
T_{x} & T_{y} & T_{z} & S
\end{bmatrix}
$$