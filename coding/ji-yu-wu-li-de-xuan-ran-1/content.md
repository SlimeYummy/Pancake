# 基于物理的渲染 1

- NaNuNoo
- 2017-01-14
- https://fenqi.io/coding/webgl-zi-yuan-guan-li/

## 辐射与光的物理学

在实现虚拟世界的光照之前，先弄清楚真实世界中的光照是如何发生的。

### 辐射能量（Radiant Energy）

光源是一个不断向外发射光子的物体，每个光子显然都带有一定的能量。光源发出的能量叫做辐射能量（Radiant Energy），单位是焦耳 J。

辐射能量表示光源在某个时间段内发射的总能量。同一盏灯开 2 小时辐射能量大于开 5 分钟。同样 1 分钟，太阳的辐射能量肯定大于灯泡。

### 辐射通量（Radiant Flux）

光源在单位时间内发射出去的能量叫做辐射通量（Radiant Flux），更通俗地叫法是辐射功率（Radiant Power），单位是瓦特 W，也可写作 J/s。

辐射功率可以理解为光源向外发射能量的速度。太阳的辐射功率大于显示器。

### 辐射通量密度（Radiant Flux Density）

光离开光源后，来考虑照射到物体表面的情况。单位面积内的辐射通量（辐射功率）叫做辐射功率密度（Radiant Flux Density），单位是 W/m^2。

对于同一粒灯泡，距离 1 米比 10 米看起来更亮，为什么？灯泡的光是向整个空间发散出去，1 米处的辐射通量密度大于 10 米处。

辐射通量密度有几个别名。在光到达物体表面时叫做辐射照度（Irradiance）、在光离开光源表面时叫做辐射出度（Radiant Exitance）。

### 立体角（Solid Angle）

在阐述接下来的概念之前，先解释立体角（Solid Angle）。立体角，单位球面度 sr。立体角是平面角在三维空间内的扩展，对比平面角来看。

平面角单位是弧度，可以理解为单位圆上对应的弧长，范围 0 - 2 π（单位园周长 2πr）。

立体角单位是球面度，可以理解为单位球上对应的面积，范围 0 - 4π（单位球面积 4πr^2）。

### 辐射强度（Radiant Intensity）

换一个角度来考量光线，光线从光源向四面八方发射。通过单位立体角的辐射通量叫做辐射强度（Radiant Intensity），单位 W/sr。

之所以引入辐射强度，是为了便于度量某一点的通量密度。点的面积是 0，无法使用辐射照度（辐射出度），但辐射强度不会距离增大而衰减，因为立体角不会随距离变化而变化嘛。

### 辐射率（Radiance）

单位立体角的辐射通量密度，叫做辐射率（Radiance），单位是 W/sr*m^2。

辐射率实际上是眼睛（或相机）看到的的物体的颜色，在实时渲染中就是像素的颜色。物体的颜色不随距离增大而变化，因为辐射率不会随距离增大而衰减。

为什么辐照度随距离增大而衰减，但是我们看到的颜色（辐射率）却不会？距离增大，物体到达视网膜的通量密度减小，同时物体在视网膜上的立体角也减小，变化正好相抵消。

## 从辐射到光

必须承认，前一小节中我刻意忽略了光与辐射。光测量与辐射测量有很大得相似性，只需在辐射测量的基础上，考虑人眼对不同颜色光感知能力得权重。

另外，光测量存在一些独有的单位，这些单位可以和辐射测量中的单位一一对应起来。下表中前 5 行为辐射测量单位，后 5 行为光测量单位。

| 名称      | 符号   | 单位                              |
| ------- | ---- | ------------------------------- |
| 辐射能量    | Q    | $J$                             |
| 辐射通量    | Φ    | $W$、$J \cdot s^{-1}$                  |
| 辐射通量密度  | ω    | $W \cdot m^{-2}$                      |
| 辐射强度    | I    | $W \cdot sr^{-1}$                     |
| 辐射率     | L    | $W \cdot sr^{-1} \cdot m^{-2}$              |
| 光能      | Q    | $Im \cdot s$                          |
| 光通量     | Φ    | $Im$（流明）                        |
| 照度、光出射度 | E、M  | $lx$（勒克斯）、$Im \cdot m^{-2}$           |
| 发光强度    | I    | $cd$（坎德拉）、$Im \cdot sr^{-1}$          |
| 亮度      | L    | $cd \cdot m^{-2}$、$Im \cdot sr^{-1} \cdot m^{-2}$ |

## 光照方程

通常，实时渲染中只考虑光的反射现象。即光源发出光，在物体表面发生反射最终到达肉眼。忽略了次表面散射，半透明物体投射得现象。

(1)    $L_{0} (v) = \int _ \Omega \Phi (l, v) \otimes L_i (l) (n \cdot l) d \omega _ i$

公式 (1) 是仅考虑反射现象得光照方程，是实时渲染的起点。$l$ 是入射光向量，$v$ 是反射光向量，$ρ(l, v)$ 是 BRDF，$L_i(l)$ 是入射光对颜色得贡献，$(n \cdot l)$ 是入射光向量乘表面法向量。

实际编程时，一般会用求和形式代替积分，写作：

(2)    $L_{0}(v) = \sum^{n}_{k=1}ρ(l_{k}, v){\otimes}L_i(l_{k})(n \cdot l_{k})$

可以理解为，当前点的颜色取决于所有光源对改点贡献的叠加。

假设场景中只存在单一光源，还可以简化为式 (3)，看起来清爽多了，后面都以单一光源举例。

(3)    $L_{0}(v) = ρ(l, v){\otimes}L(l)(n \cdot l)$

## BRDF 相关概念

在介绍 BRDF 前先理清楚 Lamert、Blinn-Phong、BRDF、物理正确的 BRDF，这几者之间的关系。

Lambert 与 Blinn-Phong 是 3D 开发者最先接触到的光照模型。资料上通常先介绍 Lambert 与 Blinn-Phong，紧接着介绍 BRDF。初次阅读时可能会以为 BRDF 也是一种光照模型，实际上。实际上 BRDF 是高一层级的概念，Lambert 与 Blinn-Phong 都属于 BRDF。将 BRDF 理解为一类光照模型的总称即可，这类光照模型只考虑光线反射，不考虑散射与透射。

Blinn-Phong 是一种 BRDF，但 Blinn-Phong 显然不是物理正确的（否则就没有这篇文章了）。物理正确的 BRDF 需要符合一些额外条件。

## 漫反射的 BRDF

物理正确的漫反射 BRDF 比较简单，从 Lambert 入手可以推导得到。

$c_{lambert}=c_{diffuse}{\otimes}c_{light} \cdot (l \cdot v)$

Lambert 光照方程都写过，$c_{diffuse}$ 表面漫反射颜色，$c_{light}$ 光源颜色，$(n \cdot l)$ 入射光向量乘表面法向量。

将式 () 与 式 (3) 对比。$(n \cdot l)$ 部分两式都有。 $c_{light}$ 物理意义上与 $L_i(l)$ 相同。剩下 $ρ(l,v)$ 与 $c_{diffuse}$ 部分。因此，原版的 Lambert 光照方程中，BRDF 就是 $c_{diffuse}$，没错是个颜色常量。

$ρ(l, v) = c_{diffuse}$

遗憾的是，这个方程。简单，除一个 π 吧，得到这个方程，它就是物理正确的漫反射 BRDF。

$ρ(l, v) = {c_{diffuse}}/{π}$

## 微表面理论

$L_o(\mathbf{v})=\pi\rho(\mathbf{l_c},\mathbf{v})\otimes\mathbf{c}_{light}\mathbf{n}\cdot\mathbf{l_c}$

$\rho(\mathbf{l},\mathbf{v})=\frac{F(\mathbf{l},\mathbf{h})G(\mathbf{l},\mathbf{v},\mathbf{h})D(\mathbf{h})}{4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})}$

其中 $F(\mathbf{l}, \mathbf{h})$ 是菲涅尔项，$G(\mathbf{l},\mathbf{v},\mathbf{h})$ 是微表面反射的比例，$D(\mathbf{h})$ 是法线分布函数，$4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})$ 是个校正因子。

$F_{Schlick}(\mathbf{c}_{spec},\mathbf{l},\mathbf{h})=\mathbf{c}_{spec}+(1-\mathbf{c}_{spec})(1-\mathbf{l}\cdot\mathbf{h})^5$

菲涅尔项一般用式 () 来近似。

## 镜面反射的 BRDF

单一精确光源的渲染方程。

$L_o(\mathbf{v})=\pi\rho(\mathbf{l},\mathbf{v})\otimes\mathbf{c}_{light}(\mathbf{n}\cdot\mathbf{l})$

漫反射与镜面反射的 BRDF 是不同的需要分别计算再加和。

$L_o(\mathbf{v})=\pi(\rho_{diff}(\mathbf{l},\mathbf{v})+\rho_{spec}(\mathbf{l},\mathbf{v}))\otimes\mathbf{c}_{light}(\mathbf{n}\cdot\mathbf{l})$

漫反射：

$\rho_{diff}(\mathbf{l},\mathbf{v})= \cfrac{\mathbf{c}_{diff}}{π}$

镜面反射：

$\rho_{spec}(\mathbf{l},\mathbf{v})=\cfrac{F(\mathbf{l},\mathbf{h})G(\mathbf{l},\mathbf{v},\mathbf{h})D(\mathbf{h})}{4(\mathbf{n}\cdot\mathbf{l})(\mathbf{n}\cdot\mathbf{v})}$

F 菲涅尔项使用：

$F_{Schlick}(\mathbf{c}_{spec},\mathbf{l},\mathbf{h})=\mathbf{c}_{spec}+(1-\mathbf{c}_{spec})(1-\mathbf{l}\cdot\mathbf{h})^5$

G 项、D 项有很多选择，它们是不同 BRDF 的主要差异。以当前流行的 GGX 为例。

$D_{GGX}(\mathbf{v}) = \cfrac{\alpha^2}{\pi((\mathbf{n}\cdot\mathbf{m})^2 (\alpha^2 - 1) + 1)^2}$



## 简化的环境光

## 材质系统



