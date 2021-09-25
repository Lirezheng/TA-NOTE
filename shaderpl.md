---
typora-copy-images-to: ue4\TA-NOTE
---



本文是学习《shader入门精要》和《计算机图形学》和《shader lab开发实战》的总结篇1，图源来自书中。



# 图形管线部分

## 前置了解

一些图形学史：

​	1962年，MIT林肯实验室的Sutherland在其论文中首次使用“Computer Graphics”术语。

​	同年，法国雷诺公司的工程师Bezier提出贝塞尔曲线并用于曲面设计。

​	1964年，MIT的Coons 提出“超限插值”思想，通过插值4条任意边界曲线构造曲面。

​	CAD：Computer Aided Design，是计算机辅助设计简称，60年代已经出现。

​	70到80年代光照模型发展迅速。70年，Bounknight提出第一个光反射模型。

​	71年，Gourand提出“漫反射模型+插值”思想，又称Gourand明暗处理。

​	75年，Phong提出著名光照模型——“Phong模型”。

​	80年，Whitted提出光透视模型——“Whitteed模型”，并给出光线追踪算法示例。

​	由于真实感模型绘制周期长，实时图形学技术开始用于某些实时显示的任务。

​	实时图形技术：通过损失一定图形质量达到实时绘制图像的目的。

​	LOD：Level of detail，层次细节显示和简化技术，降低三维模型的复杂度。    

------

DirectX和openGL：

​	是什么：图像编程接口，不是语言，是底层硬件的一种抽象。

​	用处是：可以使用他们提供的接口渲染图形。

​	原理是：向接口发送渲染命令（Draw Call），接口向显卡驱动发送渲染命令，显卡驱动翻译命令，GPU绘制。

​	               

------

GPU：图形处理单元 ；图形处理器。   

​			主流图形硬件供应：NVIDIA,  AMD。

​			支持：DirectX和openGL图形标准。

​			特点：高并行性，可编程性。

​			用处：图形处理，科学计算。

------

HLSL,GLSL,CG:高级着色语言。

​		是什么：图形编程接口的上层抽象。

​		应用：可编程着色器阶段。

​		HLSL：应用于DirectX平台。

​		GLSL：应用于openGL平台。

​		CG：跨平台，能编译成中间语言。

现在主流使用的是CG/HLSL语言，由Microsoft维护。

  

------

重要概念：

图元：渲染所需的几何信息，可以是点，线，三角面等。     

纹理：提高真实感，通常纹理是二维的。用法是把一幅图像贴到物体表面。 

插值：用来填充[图像变换](https://baike.baidu.com/item/图像变换/807964)时像素之间的空隙，即从离散到连续的重构。在图形学中，插值对象可能是纹素、动画关键帧等。

LOD：Level of detail，层次细节显示和简化技术，降低三维模型的复杂度。 

NDC（Normalized Device Coordinates）：归一化设备坐标。

MVP矩阵：多个转换矩阵串联成一个矩阵，用于将顶点从模型空间中转换到裁剪空间。

坐标空间：模型空间，世界空间，相机空间，裁剪空间，归一化设备空间，屏幕空间。           

------

​                                                                                                                                                                                                                                                                                                                  

​	

## 渲染管线初见：

### 1.概念：

![image-20210417174249219](pictures.assets/image-20210417174249219.png)

![image-20210417174315048](pictures.assets/image-20210417174315048.png?lastModify=1627884420)

**工作任务**：由三维场景（数据）出发，生成二维图像。

①应用阶段负责：将数据以图元形式提供给图形硬件。

1. 准备场景数据（摄像机视锥，模型顶点及法线信息等，光源位置及类型等）。
2. 提高渲染性能，例如做粗粒度剔除工作。（把看不到的先裁掉）。
3. 设置模型渲染状态（材质，纹理，shader等）。

应用阶段还包含Frustum Culling（视锥体裁剪）。一般是由CPU检测包围盒（或在DX11后使用GPU的Compute Shader完成），以决定是否将某个物体的信息装配好图元发给GPU的Vertex shader。该操作不会裁剪顶点，仅决定是否发送物体的信息。

GPU中没有物体概念，它的工作是以图元为单位的，因此剔除这些图元的效率不高。采用视锥体裁剪，能减少GPU的渲染任务，将不在视锥内的物体剔除，从而增加渲染性能。该操作一般无需干预。

![image-20210803010454021](pictures.assets/image-20210803010454021.png)

除此之外CPU还能使用Occlusion Culling（遮挡剔除的技术），与视锥体裁剪共同作用，以减少遮挡物体后续的多余交互，减少GPU的工作。UE中采用回读深度图进行该操作。

**该阶段最重要的是**：将需要绘制的几何体输入到管线的下一阶段，即将绘制图元显示在屏幕上。



②几何阶段负责：处理绘制的几何相关事宜。

1. 对象：渲染图元。
2. 操作：逐顶点，逐多边形操作，将图元从三维坐标变换为二维屏幕坐标。
3. 输出：屏幕空间二维顶点坐标，包含深度，着色等信息。

管线中的一个功能阶段可以划分成多个更小的管线阶段，不一定是单个管线阶段。

该阶段重点要对若干坐标系有所认识，并了解之间的转换关系：

1. 模型坐标系
2. 世界坐标系
3. 摄像机坐标系
4. 裁剪坐标系
5. 归一化设备坐标系
6. 屏幕坐标系





③光栅化阶段负责：给每个像素配色，以便正确绘制图像。

1. 对象：上一阶段输出的图元数据。
2. 操作：决定图元中像素去留，对逐顶点数据插值，再逐像素处理，把屏幕空间二位顶点坐标转化为屏幕上像素。
3. 输出：产生像素，渲染二维图像。







### 2.几何与光栅化的详要





![image-20210417174315048](pictures.assets/image-20210417174315048.png)



#### 几何阶段：

①顶点着色器是我们的重点，作用是

1. ​		实现顶点空间变换(主要是把顶点坐标从模型空间变换到齐次裁剪空间下，利用MVP矩阵)，这是VS至少要进行的操作。
2. ​		顶点附属数据的计算（将装配阶段输入的图元信息进行再次处理，得到法线，颜色，纹理坐标，或进行顶点变换等），如果采用GPU实例化技术，那么还会在VS中处理实例ID，以便材质渲染时能正确访问特定的着色信息。

![image-20210505171524755](pictures.assets/image-20210505171524755.png)

​									顶点着色器中应用MVP矩阵	

②曲面细分着色器（可选）作用：	细分图元，一般不会选择在移动平台中使用。在UE4中有Tessellation项供曲面细分。细化后增加的顶点会参与VS的计算，但不算在输入装配阶段的顶点。

细化阶段分为三个着色器：

1. 外壳着色器：可编程。在控制点阶段和修补程序常量阶段运行，可输出细化控制点及相关的各个常量以供后续计算。
2. 细化着色器：创建用于域着色器的采样方式，并生成较小的图元。
3. 域着色器：细化后输出顶点位置，并完成细化后的顶点计算。

③几何着色器（可选）作用：	可编程，可以对当前图元进行调整、创建、删除或操控它们之间的关系。该阶段可以处理各种图元及其相邻点。

该阶段一些常见的算法：Sprite的扩展，动态粒子系统，皮毛生成等。

④裁剪（可配置）作用：

1. 剔除掉不在摄像视野内的顶点。
2. 裁剪边缘三角图元，并使用新顶点代替（使用四维齐次坐标进行插值）。

![image-20210505170033876](pictures.assets/image-20210505170033876.png)



⑤屏幕映射作用：将四维齐次坐标归一化得到三维NDC坐标，其中输入的z坐标会进入到“**Z缓冲**”中，而（x，y）从[-1,1]映射到二维屏幕坐标系下。

不同的图形标准下最小窗口坐标值（0，0）在屏幕坐标系中会有差异。GL在左下，DX在左上。

NDC中z分量：openGL的坐标范围[-1,1]; DirectX的坐标范围[0,1]。





以上顶点坐标变换步骤（从模型空间到世界空间到相机空间再到裁剪空间）通常我们直接用MVP矩阵求得。

从裁剪空间到归一化设备空间再到屏幕空间的操作：齐次除法和屏幕映射通常在图形API中进行。



------

#### 光栅化阶段：

①三角形设置：计算光栅化一个三角网格需要的信息。

②三角形遍历：检查每个像素是否被一个三角网格覆盖。若被覆盖，生成一个片元。生成片元中的状态是对3个顶点的信息获得的。

③片元着色器：根据插值后得到的片元信息，计算输出该片元的颜色（仅影响单个片元）。

​	重要使用之一——纹理采样。顶点着色器输出每个顶点的纹理坐标，经过步骤②的插值，就能得到覆盖的片元的纹理坐标。

④逐片元操作（又称输出合并Mergering阶段）：

1. ​	进行一些测试如深度测试，模板测试，透明度测试。

2. ​	混合操作（可配置的如Blend one one 这种操作）。

   

在unity shader中出现如下代码都是输出合并处理。

```hlsl
Blend SrcAlpha OneMinusSrcAlpha

ZTest Off

ZWrite Off

Stecil{

}
```

深度测试：

- ​	通过比较片元着色器产生深度值和z-buffer中的值,将更小的值留在z-buffer中。当然，也可以选择关闭深度写入，而只进行测试。



混合操作（很像ps图层属性）作用：

- 将source color 与destination color混合，即把片元着色器产生颜色与color-buffer混合。

  

混合阶段会丢弃不可见的片元浪费计算资源，因此现代GPU还有early-z功能，提前测试可见性以减少计算。



最终会将**颜色缓冲**中的内容输出到**帧缓存**中，显示器会按照一定频率读取其中的内容并输出。

关于延迟渲染和前向渲染。

前向渲染。

- 在**顶点着色器阶段**就进行光照着色，并在光栅化阶段对着色信息进行**自动插值处理**。每个**非平行光源**将执行额外的渲染通道，且不可合并批次渲染。
- UE：https://docs.unrealengine.com/4.26/zh-CN/TestingAndOptimization/PerformanceAndProfiling/ForwardRenderer/

延迟渲染。

- 会收集多张贴图信息（基色，高光，法线图，深度图等），使光照效果叠加，与前向渲染对光源数量的处理对比减少了额外的开销。但阴影绘制仍与光源数量有关。
- 延迟渲染必须要能支持MRT（Multiple Render Target），SM3.0+标准和深度渲染纹理等。因移动端GPU的渲染特性局限，所以在移动端不会使用延迟渲染，通常以前向渲染为主。延迟渲染仅支持透视视角。
- 基于Gbuffer的像素对应特性，不支持MSAA。但可以使用基于后处理的抗锯齿和DSR（Dynamic Super Resolution）。

 

# 图形渲染与视觉

## 问题先导：

1.问：渲染中如何决定**像素的光照计算**？

答：使用光照模型。



2.问：为什么我们能看到并判断物体的颜色？

答：进入人眼（摄像机）的**光的波长**影响。

------



## 光源

### 光源量化：

实时渲染中，光源通常为点光源，那么光源发出光的多少如何量化呢？

- 我们使用辐照度（irradiance）：单位时间内穿过单位面积的辐射通量。



------

### 光线和物体的作用方式：

光与物体会相交，相交结果有：散射（scattering），吸收（absorption）。

- 散射：只会改变光的方向。存在于不同光学性质的两者物质分界处。散射到外部的叫反射（reflection）；散射到内部的叫折射（refraction）。
- 吸收：只会改变光量（光的颜色和密度）。存在于物种的内部。

- 折射进入内部的光，一部分又会重新射出物质表面，而另一部分会被物质直接吸收。

在光照模型中，对两种散射方向的区分计算：

- 高光反射（specular）：物体表面反射的光线。
- 漫反射（diffuse）：如下图除反射外的其它光线作用。

![image-20210510154237619](pictures.assets/image-20210510154237619.png)

------

## 标准光照模型

### Phong模型：

由Phong提出的模型基本理念：

- 标准光照模型只关心直接光照，即一次反射光照。
- 间接关照：多次反射后进入摄像机内的光照



模型将进入摄像机内部的光线分为4个部分：

- 自发光（emissive）：没有使用全局光照的情况下，物体本身对某个方向发射一定的辐射量，但不会对环境造成影响。

- 高光反射（specular）：光线在完全镜面反射时散射的辐照度。

- 漫反射（diffuse）：光源照射到物体表面，向各个方向散射的辐照度。

- 环境光（ambient）：其它间接光照。

**环境光**：通常是个常量，计算公式如下
$$
c_{ambient} = g_{ambient}
$$
**自发光**：光源直接进入摄像机，直接使用材质的自发光颜色：
$$
c_{emissive} = m_{emissive}
$$
**漫反射**(Lambertian着色)：与视角方向无关，与物体粗糙度有关。

![image-20210510200758587](pictures.assets/image-20210510200758587.png)


$$
c_{diffuse} = (c_{light} · m_{diffuse})max(0,\vec n ·\vec l)
$$
**高光反射**：高光反射是与视角有关的，视角与光反射的方向的夹角θ决定我们看到高光的“量”是多少。

![image-20210417174948403](pictures.assets/image-20210417174948403.png)

反射方向计算公式：
$$
\vec r = 2(\vec n·\vec l)\vec n - \vec l 
$$
Phong模型高光计算公式：
$$
C_{specualr}= ( C_{light} · m_{specualr} ) max ( 0 , \vec v * \vec r)^{m_{gloss}}
$$

------



### Blinn—Phong模型

但是，Phong模型中反射方向计算过于困难，通常我们使用Blinn—Phong模型，避免计算反射方向，而改为求解矢量h。
$$
\vec h = \frac{\vec v + \vec l}{|\vec v + \vec l|}
$$


![image-20210417175001286](pictures.assets/image-20210417175001286.png)



为什么求解h矢量替代且更方便：

- ​	观察Blinn模型，我们能看到最多高光的位置在光线反射的方向上，此时视角与光线的角分线（h矢量）正处于法线的位置，因此何不用法线方向与h矢量之间的夹角计算高光值呢？

Blinn—Phong模型高光计算：
$$
C_{specualr}= ( C_{light} · m_{specualr} ) max ( 0 , \vec v * \vec h)^{m_{gloss}}
$$








### 着色（shading）方式：



- 平滑着色（Flat shading）:一个三角面片用同一颜色。
- 逐顶点（Gouraud shading）：在顶点着色器中计算。每顶点计算光照，在渲染图元内部进行线行插值，最后输出像素颜色。对于强高光反射，可能产生失真。
- 逐像素(Phong shading)：在片元着色器中计算。以每个像素为基础，得到法线（顶点法线插值得到，或法线纹理采样得到），然后再进行光照模型的计算。又称法线插值着色技术。**注意：顶点着色器到像色着色器间插值操作会改变法线等的值，因此在fragment中需要归一化。**

![image-20210510234649734](pictures.assets/image-20210510234649734.png)

以上结果不一定代表着色方式的好坏——当三角面片足够多时，他们的着色效果其实差不多，但平面着色此时性能上更优。



------

Blinn—Phong模型的局限：

- ​	各项同性：固定视角和光源方向，并旋转表面，反射不会发生任何变化。就是说光与物体各个表面的交互结果是一样的。

以上光照模型都有局限（如Blinn—Phong模型）：

<img src="pictures.assets/image-20210511000835938.png" alt="image-20210511000835938" style="zoom:50%;" />

我们可以看到物体背光区域是全黑的，几乎没有模型细节，如曲面的一些细微变化。而半兰伯特模型在此基础上对漫反射计算有了改善。

### **半兰伯特模型**：

广义半兰伯特光照模型公式：
$$
c_d=(c_l * m_d)(α(\vec n*\vec l)+β)
$$

*大多数情况下，α和β都是0.5.*

改良原理：

- 通过设值α和β为0.5，将结果范围从[-1,1]映射到[0,1],将法线与光线之间的夹角大于90°的情况（背光）也考虑进漫反射的计算中，因此背光面也有了明暗变化。

  <img src="pictures.assets/image-20210511001937439.png" alt="image-20210511001937439" style="zoom:50%;" />

------

思路较为清晰的PBR文章：

https://zhuanlan.zhihu.com/p/66518450

## BRDF

英文：Bidirectional Reflectance Distribution  Function.

![BRDF定义](pictures.assets/image-20210510191243391.png)

用以描述某点如何反射光线的方程，通常BRDF是如上的数学公式。

其中l是入射光方向，v是出射方向，给定两个方向来描述光是如何在物体表面进行反射的。

BRDF模型有本节涉及的经验模型，还有其它如基于物理的模型。

- 经验模型：对真实场景的理想简化，不能准确反映光与物体的交互，但渲染中效果不错。
- 基于物理的模型：真实模拟光与物体的交互。



### PBR（基于物理的BRDF）

找了很多文章，特别是逼乎的，几乎每篇都有不同点，公式也有错，几篇通读下来，反而脑子不好使了。我去。

PBR，基于物理的渲染，图形学中用数学建模的方式模拟物体材质与各种散射光交互从而渲染更加自然的外观。PBR可以在任何照明环境中完美工作。注意，PBR同样是光线实际情况的近似。

实时渲染中的PBR方程

![[公式]](https://www.zhihu.com/equation?tex=L_o+%3D+%5Cint_%7B%5COmega%7D%5E%7B%7D+f_rL_icos%5Ctheta_i+d%5Comega_i++%3D++%5Cint_%7B%5COmega%7D%5E%7B%7D+%28k_d+%5Cfrac%7B+c%7D%7B%5Cpi%7D+%2B+k_s%5Cfrac%7BDFG%7D%7B4cos%5Ctheta_icos%5Ctheta_o%7D%29L_icos%5Ctheta_i+d%5Comega_i%5C%5C+)

![image-20210805015423822](pictures.assets/image-20210805015423822.png)

​													对应物理量的符号。

#### 次表面散射

Subsurface Scattering：

![image-20210625164044255](https://i.loli.net/2021/06/25/EoCUMzhB7bwAyW3.png)

次表面散射(Sub-Surface-Scattering)简称3S，光在物体内部散射后,通过物体表面其它顶点再次射出物体并进入人的视野而产生的现象。光线最终会在物体中弥散，导致出射的光线各处都有，产生一种透亮的感觉。

适用于：需要考虑表面下的细节的材质，如人的皮肤，玉石，大理石，蜡烛等一些半透明材质。



Subsurface Profile（次表面轮廓）：

看法：非常适用于复原原画中对应的涂抹手法。

差异：Subsurface Profile实现与Subsurface有区别。前者运用了Screen Space，即屏幕图像处理的方法进行模糊，以此达到次表面散射的效果。其通过调节Scatter Radius（散射半径）进行图像偏移以此模糊图像。

https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/Materials/HowTo/Subsurface_Profile/

#### 菲涅尔反射

![image-20210804144829333](pictures.assets/image-20210804144829333.png)

作用：凸显人物轮廓，能量罩常用，水面材质常用。

**菲涅尔方程**（(Fresnel Reflection)），用于描述光在**不同折射率的介质之间**的行为。由该方程推导出的光的反射称为“菲涅尔反射”。参考下图，菲涅尔方程就是描述光经过两介质界面时，反射和透射的光强比重。

​											             θi入射角、θt透射角。

<img src="pictures.assets/v2-5b458ba05cceb8eb94c257fb93f77de1_r.jpg" alt="preview" style="zoom:50%;" />

​														

Fr是**菲涅尔系数**,也叫**光线反射比例**，它是针对**高光部分的**，也即**完全镜面反射**。菲涅尔系数满足 ![image-20210804154405618](pictures.assets/image-20210804154405618.png)

通常，Fr可以直接由麦克斯韦方程组推导，又渲染中使用非偏振光，因此可以得到              

​                                               ![image-20210805175225963](pictures.assets/image-20210805175225963.png)      

在不同观察角度下，使用不同材质求得菲涅尔系数。在观察角度接近掠夺角(90度)的时候, 菲涅尔系数都是趋近于1的, 这种现象叫做**菲涅尔现象**。

<img src="pictures.assets/image-20210805175521373.png" alt="image-20210805175521373" style="zoom:50%;" />

可我不会上述推导啊！幸运的是，大神帮我们把Fr的Schlick近似公式整好了，这就是我们常用的**菲涅尔方程**：
$$
F_r= F_0 + (1-F_0)pow(1- dot( N, V ),5 )
$$




UE4 菲涅尔BRDF实现，取代了pow更有效率：

![image-20210806013847346](pictures.assets/image-20210806013847346.png)

```c++
float3 F_Schlick( float3 SpecularColor, float VoH )
{
    //原公式代码
    // float dotEH = saturate(dot(E, H))
	//FresnelTerm = SpecularColor + (1.0f - SpecularColor) * pow(1 - dotEH, 5);
    
    
    
    
	float Fc = Pow5( 1 - VoH );	// 1 sub, 3 mul
	//return Fc + (1 - Fc) * SpecularColor;// 1 add, 3 mad
	
    // Anything less than 2% is physically impossible and is instead considered to be shadowing
	return saturate( 50.0 * SpecularColor.g ) * Fc + (1 - Fc) * SpecularColor;
	//ShadingModelContext.SpecularColor = 0.04;     
	//SpecularColor就是F0.
}
```

UE4菲涅尔节点实现：

```c++
half Fresnel(half Exponent, half BaseReflectionFraction) 
{
    half NoV 	= max(dot(N, V), 0.0f);
    half Fres 	= pow(abs(1 - NoV), Exponent);
    Fres        = Fres * (1 - BaseReflectionFracetion) + 		BaseReflectionFraction;
    return Fres;
}
```

UE中的菲涅尔节点，光照基于摄像机角度形成不同强度反射的现象。

![image-20210726020102741](file://C:/Users/pc/Desktop/shader/ue4/UE4%E6%9D%90%E8%B4%A8%E7%A7%AF%E7%B4%AF%E5%9B%BE%E7%89%87/image-20210726020102741.png?lastModify=1628059600)
$$
Fresnel_{Node}= pow(1- dot( Surface_{Nnormal}, Camera_{Direction} ),5 )
$$
https://seblagarde.wordpress.com/2012/06/03/spherical-gaussien-approximation-for-blinn-phong-phong-and-fresnel/

##### 电介质

在菲涅尔方程中，不管是 Fr 还是 F0都是一组**RGB三通道**的值。而实际上，电介质的菲涅尔系数不会随着光线波长而剧烈变化，因此对于电介质来说，菲涅尔系数我们通常都用单通道值来表示。以下是一些参考值：

![image-20210805180739330](pictures.assets/image-20210805180739330.png)



**观察角为0时**的菲涅尔系数记为 F0（**BaseReflectFraction**，反射分布，**法向入射的高光/镜面反射率**），将两种介质的折射率比例记为 n得到:

![image-20210806021545355](pictures.assets/image-20210806021545355.png)

一般，我们会直接给定F0一个特定值。由于电介质的F0值都较小，所以在实时渲染中，一般取0.04为默认值。









##### 金属

金属折射率是复数，其折射率表示(k为吸收率)：
$$
\vec{η} = η + ik
$$
对于金属来说，它的折射率会随着光线波长而剧烈变化，因此在Schlick近似公式中，F0要用一组**RGB三通道**的值表示。下图是一些金属的F0值：

![image-20210805183112949](pictures.assets/image-20210805183112949.png)

另外，金属内部都是可以运动的电子，折射进金属内部的光线都会被吸收，因此，金属不会产生漫反射。

实际渲染中，物体是可能同时存在金属和非金属的部分，因此我们需要定义一个值，来确切地表示金属在其中的比例，那就是——“**Metallic**”，**金属度**。

同时我们定义金属的F0为**Albedo**（也可作为金属的表面颜色理解），电介质的F0为0.04，那么，同时表示电介质和金属的F0值，我们可以用线性插值求得：
	
$$
F0 = Metallic * Albedo + (1-Metallic) * [0.04,0.04,0.04]
$$


最后我们再厘清两个概念，Fr是关于多少的光被反射的概率，F0是关于反射光的辐射能量占总辐射能量的百分比。



#### 微平面

![image-20210805020911681](pictures.assets/image-20210805020911681.png)

微平面理论（Microfacet），假设表面由不同方向的微小平面组成，每一个微小平面如上图，都会有各自的反射。微平面针对的是**高光部分**模拟修正。

Half vector ，“半角向量”就是指能反射到视线的法线。当半角向量指向视线时，我们才能看到某个微表面。

当然，我们不可能将每个微平面的法线取出来计算光照。实际上，我们可以用“**法线分布函数**”NDF来求得，半角向量指向视线的微平面占总微平面的比例。再用这个比例，修正高光强度，就可以模拟出微表面。

NDF（该NDF为Trowbridge-Reitz GGX）：

![image-20210805022344469](pictures.assets/image-20210805022344469.png)

其中，α即为**粗糙度**（**Roughness**），取值范围为0-1。也就是说，我们在材质面板中输入的Roughness，其实质是作为NDF的输入，从而影响材质表面光照。



又由于部分入射光，或者反射光会被遮挡，尽管Half vector确实指向视线，但是却没有实际作用。因此需要定义一个**遮挡函数G**，给定一个方向，就能算出该方向上微表面自遮挡的比例，从而进一步限制微表面反射得到的高光强度。

![image-20210805023719868](pictures.assets/image-20210805023719868.png)

Schlick-GGX近似函数：

![image-20210805024941971](pictures.assets/image-20210805024941971.png)

考虑入射和反射同时遮挡，即得：

![image-20210805161045727](pictures.assets/image-20210805161045727.png)

省略推导过程，得到最终得遮挡函数（Geometry ）近似：


![image-20210805162056546](pictures.assets/image-20210805162056546.png)

BTW:我们常看到高光反射有点模糊，是因为有些反射光照十分接近完全镜面反射。





#### 能量守恒

关于能量守恒，在渲染中，指的是**入射光的能量**要与**反射光、折射后再出射以及被内部吸收的总能量**要相等。

![image-20210805163654671](pictures.assets/image-20210805163654671.png)

光照效果，通常分为Diffuse和Specular。Diffuse，反射的没有方向性的光。Specular，镜面反射的光。如果Specular的光多，那么说明折射的光少，产生的Diffuse就少。

因此，我们需要根据能量守恒，为我们的Diffuse和Specular再做进一步修正，即

$$
L=k_{diff} * Diffuse + k_{spec}*Specular
$$
接下来，就正式介绍完整的公式说明。



#### 常见的BRDF方程

![image-20210806022303859](pictures.assets/image-20210806022303859.png)

Cook-Torrance模型：

![image-20210806022558946](pictures.assets/image-20210806022558946.png)

其中fd和fs又是关于diffuse和specular的BRDF方程。



##### 漫反射BRDF

我们常使用Labertian模型来描述漫反射，它假定物体表面光的亮度不随视角改变，这样的光叫漫反射。

fd (Lambertian模型Brdf)：

![image-20210806015850467](pictures.assets/image-20210806015850467.png)

在UE的论文中，epic评估了Burley和Lambertian漫反射模型的差异，尽管Burley开销大，但却并没有更好的表现。较复杂的模型反而不能有效利用基于图像或球谐的光照。

该模型中，c_diff是漫反射的反照率。

反照率（albedo）与反射率（reflection）的区别

albedo：用来表示全波段的反射能量与入射能量之比。

reflection：用来表示某一个波长的反射能量与入射能量之比。



##### 高光BRDF

![image-20210806023220836](pictures.assets/image-20210806023220836.png)

在UE中分别是：
$$
k_s = F
$$
菲涅尔。

![image-20210806023306978](pictures.assets/image-20210806023306978.png)

微表面分布。

![image-20210806023321523](pictures.assets/image-20210806023321523.png)

几何函数（Geometry function）。

![image-20210806023336866](pictures.assets/image-20210806023336866.png)



再回眸：

![image-20210806023707716](pictures.assets/image-20210806023707716.png)
$$
k_d+k_s =1
$$


#### PBR方程总结

反射方程

![image-20210806021956127](pictures.assets/image-20210806021956127.png)



PBR方程

![image-20210805170347637](pictures.assets/image-20210805170347637.png)

由菲涅尔反射，我们知道F本身表示为反射比例，也即高光反射比例。所以对于漫反射而言，非金属部分，又不是高光反射的比例就是kd。
$$
k_d = （1-F）（1-Metallic）
$$


实际渲染的PBR方程：

![image-20210805170356239](pictures.assets/image-20210805170356239.png)







https://zhuanlan.zhihu.com/p/31534769

https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/Materials/HowTo/Fresnel/

https://zhuanlan.zhihu.com/p/158025828

### 基于图像照明（IBL）

基于图像照明（Image-Based Lighting, IBL）









## 抗锯齿方法

本章是只列虚幻相关的抗锯齿方法。

Nvidia中使用抗锯齿的示例图：

<img src="pictures.assets/image-20210806150710366.png" alt="image-20210806150710366" style="zoom:80%;" />



AA：Anti-aliasing，抗锯齿，指对锯齿状线条的平滑处理。



### SSAA

SSAA：超级采样抗锯齿（Super-Sampling Anti-Aliasing，简称 SSAA），早期抗锯齿方法，较为消耗资源。工作方式：将图像映射到缓存，放大，采样2个或4个邻近像素混合然后作为单个像素值，最终将所有处理后的图片还原到原大小，并替代原图像保存到帧缓存中。

采样方法：

- OGSS，顺序栅格超采样，采样选2个邻近像素。
- RGSS，旋转栅格超采样，采样选4个邻近像素。



### MSAA

MSAA：Multi sampling Anti-Aliasing，多重采样抗锯齿，特殊的超级采样抗锯齿（SSAA）。工作方式：对**Z缓存和模板缓存**中的数据进行SSAA。比SSAA效果弱，但消耗大减。由于其工作方式的原因，不可在延迟渲染中使用。



### FXAA

FXAA：快速近似抗锯齿(Fast Approximate Anti-Aliasing，简称 FXAA) ，MSAA高性能近似。单纯的后期处理着色器，运行于后期处理阶段。不依赖于显卡。所以FXAA在UE这一注重延迟渲染的引擎中颇受重用。



### TAA

TAA：时间性抗锯齿（Temporal Anti-Aliasing，简称 TXAA），将MSAA、时间滤波、后处理结合，呈现高保真的视觉效果。相比于MSAA，TXAA能呈现更平滑的效果。同时，TAA能对场景进行抖动采样以减少闪烁的情况。TAA在UE中是默认开启的。

关于性能：TXAA 2X 和 TXAA 4X。TXAA 2X 可提供堪比 8X MSAA 的视觉保真 度，然而所需性能却与 2X MSAA 相类似；TXAA 4X 的图像保真度胜过 8XMSAA，所需性能 仅仅与 4X MSAA 相当



chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=https%3A%2F%2Fdeveloper.download.nvidia.cn%2Fassets%2Fgamedev%2Ffiles%2Fsdk%2F11%2FFXAA_WhitePaper.pdf

https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/PostProcessEffects/AntiAliasing/

https://docs.unrealengine.com/4.26/zh-CN/Resources/ContentExamples/PostProcessing/1_14/



# 纹理贴图

纹理贴图（Texturing），使用图像、函数等数据来源修改物体表面外观。

## 纹理管线与过程展示



![image-20210827152841867](pictures.assets/image-20210827152841867.png)

![image-20210827152849702](pictures.assets/image-20210827152849702.png)

几个概念：

- 纹素（Texel）：图像纹理中的像素。
- 贴图（Mapping）：将投影方程作用于空间中的点，并得到有关纹理的一组数值——参数空间值。
- UV：从图像上看就是物体空间坐标**投影**到参数空间中对应的值域为0到1之间的坐标。

纹理是如何被我们看到的：

- 观察某帧中的一个位置（x,y,z）,该位置坐标是基于物体空间的。
- 将该坐标使用某一投影函数转换到参数空间下的uv坐标。
- 使用映射函数，将uv坐标转换为纹理图片中的坐标。
- 利用纹理图片空间中的坐标（通常是数组的索引），查找到相应的纹素的颜色。
- 再之后，可以对查找结果进行值变换以改变纹理表现效果。
- 将新值（或原纹素值）用于着色方程，作为漫反射颜色，并替换之前的漫反射颜色。











## 凹凸贴图

![image-20210827162335457](pictures.assets/image-20210827162335457.png)

掌握要求：**简述概念，并写出相应的shader代码。**

### 凹凸贴图

改变表面光照方程的法线，或计算照明前对待渲染的像素加上一个来自高度图的扰动，从而实现凹凸的效果。

两种方法shader实现：

- 法向量法(normal map)

- 高度图扰动法线法(height map)


一般情况下，我们只用normal map，因为height map需要复杂的计算，无法像normal map一样直接给出凹凸情况。但两者结合会给出更好的凹凸效果。



### 移位贴图



### 法线贴图



### 视差贴图



### 浮雕贴图







[Relief mapping:凹凸贴图的极致_爱学术 (ixueshu.com)](https://www.ixueshu.com/document/3dc4369a761ca0d6318947a18e7f9386.html)

# 延迟渲染

## 延迟渲染概念

UE中默认使用延迟渲染（Defered Rendering），其中一些概念需要我们了解后才能高效地利用。因此着重讲解TA需要知道部分。

前向渲染，每光源逐个渲染物体，先着色，再深度测试，光照计算复杂度与光源有很大关系。当场景光源过多，迭代渲染过程部分片元的着色计算过于浪费。

延迟渲染：将所有着色计算延后处理地一种渲染方法。延迟渲染能够解决大量光源同时存在的问题。如何解决的？

延迟渲染工作方式：将所有物体先绘制到**几何缓冲区**（G-buffer），然后再逐光源对G-buffer的内容进行着色计算。也即在**三维空间**中**先执行深度测试**，，再在**二维空间**（G-buffer）中进行**着色计算**。

延迟渲染流程：

几何处理阶段。获得每个对象的信息，然后存储到G-buffer（G-buffer可有很多个）当中。

光照处理阶段。渲染一个屏幕大小的矩阵，然后使用G-buffer中的信息对矩阵中每一像素进行计算。



## G-buffer

G—buffer，全名Geometric Buffer，用于存储每个像素对应位置的各种信息如法线，位置，高光，漫反射颜色等。利用G-buffer中的信息，可以在二维空间中对像素进行光照处理。

二维空间图像信息：

![image-20210806142259188](pictures.assets/image-20210806142259188.png)



UE中G-buffer存储内容示例：

![image-20210806140629345](pictures.assets/image-20210806140629345.png)



## 延迟渲染管线



延迟渲染管线：

![image-20210806143237370](pictures.assets/image-20210806143237370.png)

前向渲染管线：

<img src="pictures.assets/image-20210806143146598.png" alt="image-20210806143146598" style="zoom: 67%;" />

对比两个渲染管线法线，两个管线在光栅化产生片元后有所区别。

其中延迟渲染光栅化后紧急着进行深度测试，再将单帧通过结果先存储到G-buffer保留，最终着色后存入帧缓存。

而前向渲染光栅化后直接进行着色，然后再执行深度测试。



## 延迟渲染性能分析



![image-20210629160145242](file://C:/Users/pc/Desktop/shader/ue4/UE4%E5%AE%9E%E6%97%B6%E6%B8%B2%E6%9F%93%E5%9B%BE%E7%89%87/image-20210629160145242.png?lastModify=1628232276)

优：

着色发生再延迟通道中，所用shader更少。

对有动态光照，大量光源场景的支持良好。

渲染表现高度稳定可视。

花里胡哨特性支持更好。

后处理效果支持好。

劣：

因开启多重渲染目标（MRT），抗锯齿方法不能应用多重采样抗锯齿（MSAA），默认使用时间抗锯齿（TAA）。

内存开销大，读取G-buffer需要较大带宽。

对半透明表面渲染支持不好。



参考：

【英文缩写】MRT

【中文翻译】多重渲染目标

【补充说明】这种技术指的是GPU允许我们把场景同时渲染到多个渲染目标纹理中，而不再需要为每个渲染目标纹理单独渲染完整的场景。



## 其它

各种Rendering Path：

- 正向渲染 （Forward Rendering）
- 延迟渲染 （Deferred Rendering） 
- 延迟光照 （Light Pre-Pass / Deferred Lighting） 
- 分块延迟渲染（Tile-Based Deferred Rendering）
-  Forward+（即 Tiled Forward Rendering，分块正向渲染） 
-  群组渲染 Clustered Rendering







# 全局光照技术

​																											（GI & Lightmass）

## 基于UE的光照概念描述。

全局光照，（Global illumination），既考虑**直接光照**（Direct Light），又考虑**间接光照**(Indirect Light)的渲染方式。其能为实际项目提供较为真实的照明效果。

- 直接光照（Direct Light），直接接收来自光源的光照。
- 间接光照(Indirect Light），经过环境反射后的光照。
- 理论上除了漫反射，反射、折射、阴影都属于全局光照的范畴。

在UE中分别提供两种方法**模拟光活动**，它们可以混合使用。

- **实时光照法**，支持动态光源的移动和交互。
- 利用储存在几何体表面应用的纹理中，**预计算或已烘焙的光照信息**。



Lightmaps，光照贴图，一种模拟光照的方法。光照贴图实现首先需要提前渲染，然后将预计算的光照结果存储到Lightmaps中。光照贴图一旦烘培完成，将无法动态调整。

光照贴图是如何应用的？通过与base color texture相乘，让最终整个纹理像在发光一样。

![image-20210630235645180](pictures.assets/image-20210630235645180.png)

Lightmass ：独立应用，用来控制光照渲染并将其烘培到光照贴图中、支持网格式分布渲染。简单地说，UE会用Lightmass将Lightmaps进行打包。

### 动态GI

屏幕空间全局光照（SSGI），原理是使用**后处理屏幕空间效果**，产生动态的间接光照。限制：光源**不能在视野外**或者**被场景物体遮挡**。适合与预计算GI结合视野。

光线追踪全局光照（RTGI），利用DXR框架和GPU Tracing。UE中的方法是

- **Brute Force** 方法模拟路径追踪器的地面真实参考状况，并且在实时渲染时的执行过程中最相似。此方法最消耗帧性能，同时也提供最精确的动态全局光照方法。
- **Final Gather** 方法使用两次扫描算法，采用分布式着色点和每像素固定数量样本，用精确度换取性能。对于需要实时性能的项目，Brute Force方法在精确度上的权衡意味着你的帧预算可以支持动态全局光照。

更具体的使用实时光线追踪：

[1]: https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/RayTracing/



### 预计算GI

UE中的烘焙系统提供两种方法使用Lightmass计算光照数据。

①基于CPU的Lightmass系统，用于预计算**具有固定**和**静止运动性的光源**的照明贡献部分。使用独立程序Unreal Swarm计算并生成光照数据，同时也会分配光照数据到构建场中。通常我们在烘焙光照时都会启用该进程。

![image-20210804021002002](pictures.assets/image-20210804021002002.png)

[1]: https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/Lightmass/
[2]: https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/Lightmass/UnrealSwarmOverview



②基于GPU的Lightmass系统，可以预计算**移动性设置为静止（Stationary）**或**静态（Static）的光源**的复杂光线交互，并将这些数据保存在生成的光照贴图纹理中，这些纹理又转而应用到场景几何体。速度与CPULM媲美，但是仍然处于测试中。https://docs.unrealengine.com/4.26/zh-CN/RenderingAndGraphics/GPULightmass/



# 阴影应用



**阴影映射技术（Shadow mapping）**

![img](pictures.assets/20160520003255489)

Shadow mapping，广泛应用的实时渲染阴影技术。涉及两个视角，光源视角与相机视角。

图上步骤

- 在**光源视角处**获得每个片元的深度信息，并设置一个投影矩阵，能让片元投影在指定的Shdow Map上。

- 在**相机视角处**将场景转换到光源空间并获得每个片元的深度信息，然后与采样shadow map得到的深度比较大小，以此判断是否在阴影。




Unity中使用一张纹理作为LUT，以帮助像素着色器计算逐像素光照。使用LUT的好处，只需要用参数值对LUT进行采样即可得到衰减值，减少计算。但也有弊端：

- 需要预处理的得到LUT，且衰减精度会受到纹理大小的影响。
- 无法直接使用公式计算出衰减值。



Unity的阴影：

传统Shadow Map技术，将摄像机放到光源的位置，将顶点变换到光源空间下，然后摄像机会将当前的深度信息渲染到阴影映射纹理中（Shadowmap）。我们可以单独使用一个Pass，渲染目标为shadowmap，避免多余的性能浪费。然后其它Pass将顶点变换到光源空间下，并利用它的xy分量查找该LUT以计算阴影。



屏幕空间渲染阴影（Screen Shadow Map），延迟渲染常用阴影技术。Pass中得到两个纹理，1.可投射阴影的光照的阴影映射纹理。2.摄像机的深度纹理。当摄像机中的某点深度信息，小于对应阴影映射纹理中的深度信息，就说明处于阴影中。不断重复该过程，最终得到一张屏幕空间中的阴影图。阴影图是处于屏幕空间的，我们使用时需要将顶点坐标从模型坐标变换到屏幕空间中，再进行采样。



阴影过程区别

- 一个物体可以接收阴影（receive）需要：Shader中对LUT（及阴影图）进行采样，然后用于光照运算。

- 一个物体可以投射阴影(cast)需要：该物体要加入光照的阴影映射纹理的计算中，以便其它物体采样相关信息。







**级联阴影贴图（Cascading Shadow Mapping）**





















# 图形渲染技术

## 图像处理（基于后处理）

### 颜色校正（Color  Correction）



调色，将现有图像每像素颜色转换到其它颜色过程。也支持更复杂的功能。

单一调色可以直接计算，更复杂的调色就需要用到颜色查找表（LUT）。颜色查找表直接存储在内存中，可以直接查询进行校正操作。



### 色调映射（Tone Mapping）

简单引入，太阳实际亮度范围是很广的，但是我们拍摄下来的照片却不能把真实的亮度正确的表达，更有可能会丢失部分细节，因此色调映射很有必要。

HDR与LDR

色调映射，意在解决将对比衰减过大的场景，转换到显示图像上，且尽量维持图像的细节和颜色等信息。

#### LUT

------

LUT-Look Up Table（查找表）：存储颜色记录和颜色变化。存储颜色偏移并在引擎中用此偏移。总颜色4096.

![image-20210701135548232](pictures.assets/image-20210701135548232.png)

![image-20210701135737906](pictures.assets/image-20210701135737906.png)



https://zhuanlan.zhihu.com/p/21983679

https://www.cnblogs.com/cjhd/p/7530440.html

### 镜头眩光和泛光（Lens Flare and Bloom）

Lens Flare，原理是**眼睛晶状体或相机的透镜**直面强光的现象。

Bloom，原理是**眼睛晶状体或相机的透镜**散光产生，从而在光源的周围产生光。散光眼的都懂。物理成因：透镜无法聚焦。在UE中的实现，找到一张明暗对比度强的照片，模糊，混合一张照片，在混合原图。

### 景深 （Depth of Field）



 DOF：Degree of Freedom（自由度）

![image-20210626011348636](https://i.loli.net/2021/06/26/3VPJykLIC8Dhlun.png)

弥散圆：焦点前后所成影像形成的圆叫弥散圆。弥散圆若直径过小，人眼不能识别，这样的弥散圆叫容许弥散圆(permissible circle of confusion)。

![image-20210626011839410](https://i.loli.net/2021/06/26/HPmSLwbt13VJayG.png)

景深：焦点前后两个容许弥散圆之间的距离叫景深。这一段距离是拍摄物体显示清晰的范围。



# 非真实感渲染



色度艺术映射（Tonal Art Map）：

使用提前生成的纹理实现风格渲染。纹理主要作用是模拟不同光照下的**漫反射效果**。每张纹理我们可以使用mipmaps方式存储，但是要整体笔触间隔保持比例相当。

实现方法，准备n张不同密度的纹理图。顶点着色器下计算 nl，并clamp在0和1之间得到系数diff。根据纹理张数n，计算Factor = diff *n后，我们将[0,1]区间映射到[0,n]。然后判断Factor位于哪个区间中，并计算权重值。在像素着色器中，我们对给个纹理进行采样，并乘对应的权重，得到最终的diffcolor。



# 面试题解答

基本问题

- dx与opengl的区别

- opengl与opengles的区别

  

图形与渲染管线。

- 菲涅尔反射
- MVP矩阵，一些平移旋转缩放的矩阵形式。二维三维的旋转缩放平移都要会。
- 描述渲染管线。
- 顶点着色器和片元着色器的介绍和区分。
- Z测试,earlyz技术。
- 裁剪算法实现。
- 什么是DrawCall
- 细分着色器的底层实现、应用场景，为什么需要细分着色器。
- CPU与GPU的问题。GPU性能问题。
- 判断点在三角内的方法（四种）。
- 颜色空间。
- BRDF方程。

DCC工作流程问题。

- 如何提高渲染效率
- PBR流程,PBR流程的理解。
- 美术资源制作、导入Unity，在unity中如何处理等问题。
- 导入高细分度的模型会对性能产生什么影响。

渲染原理与实现。

- UV原理
- 光照方程，光照模型，Phong，Blinn-Phong。
- 卡通着色，Bloom原理。
- NGUI的图集原理。
- lightmapping原理。
- tonemapping原理及代码。[Tone mapping进化论 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/21983679)
- IBL相关。
- 阴影的实现，阴影为什么这么消耗性能，shadow map渲染与实现。
- 半透明、透明贴图渲染顺序，透明渲染.
- 卡通渲染特点与怎么实现
- 如何实现unityshader/或UE的描边
- 各种卡通效果的实现，cellshading。
- 抗锯齿问题。
- Perlin noise和其它noise的实现、

关于shader。

- shader实现。
- shader是什么。
- shader代码结构。
- shader和材质的区别

具体项目中，某个效果实现的原理和应用。

- 哈游卡渲，米哈游sss，
- 真实海洋，真实头发。
- 分析渲染技术，对前沿技术的认识。
- 玩过什么游戏，分析游戏的材质表现 场景优化。

特定项目提问。

- 哈游卡渲，米哈游sss。
- 真实海洋，真实头发。
- 玩过什么游戏，分析游戏的材质表现 场景优化。











c++的基础知识，面向面经学习。

- 面向对象编程的特点。
- int a，b不增加存储单元的情况下呼唤AB
- 如何防止溢出
- c++堆、栈的区别
- Vector原理
- map和set的原理
- C++11特性
- 左值和右值
- 多态，动态多态的实现

操作系统。

- 线程与进程区别。
- 多线程的问题。
- mmu的作用。
- 页表的作用。

计算机网络

- TCP和UDP的区别。

算法。

- 列举常见排序。
- 快排原理。
- 堆排原理，复杂度。
- 动态规划。
- 图的存储方式。
- 最短路线算法有哪些。
- A*算法。
- 基础数组，算法，数据结构中哈希表的原理
- 双向链表的原理与应用
- 两个队列实现栈
- 算法的三维数组的排序问题。
- 实现环形缓冲区。
- 升序组合合并去重。
- 给一字符串找出最长不重复子序列
- 给一个有序序列，插入若干数字使相邻两个数之间的差不超过3





# shader中常用的操作与注意事项

精度选择（CG/HLSL）（性能/平台）：

```
float  最高精度  32位存储

half   中等精度   16位存储 |60000|

fixed  最低精度  11位存储 |2|
```

内置函数：

- ​	使用后得到的方向值没有归一化，所以不要忘记进行归一化后再计算。

防止点积为负

```
max（0，x）
x：参数
```

防止点积为负数用max操作或如下函数。

```
函数名：saturate(x)

参数x：用于操作的标量或矢量。

描述：把参数截取到【0，1】范围内，参数若是矢量，每个分量都进行这样的操作。

类型：float,float2,float3等
```

截取矩阵操作

```
(float3x3)矩阵名
```

计算反射方向

```
reflect(i,n)
i:入射方向。
n：法线方向。
```

次方计算

```
pow（x，n）
n：次数
```
