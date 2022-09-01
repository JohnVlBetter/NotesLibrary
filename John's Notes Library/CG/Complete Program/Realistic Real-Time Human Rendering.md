# 真实感人物渲染
## 真实感皮肤渲染
### GPU Gems 3 -  Advanced Techniques for Realistic Real-Time Skin Rendering
#### 对于皮肤实现的基于物理的BRDF模型
&nbsp;&nbsp;&nbsp;&nbsp;每条光线总的镜面反射为:
```hlsl
specularLight += lightColor[i] * lightShadow[i] * rho_s * 
	specBRDF( N, V, L[i], η, m) * saturate( dot( N, L[i]));
```
&nbsp;&nbsp;&nbsp;&nbsp;其中不变的roh_s项用于缩放镜面反射强度，还有表面法线N、摄像机方向V、光线方向L、折射序号η及粗糙度m。

&nbsp;&nbsp;一个用于计算镜面BRDF中Fresnel反射的函数:
```hlsl
float fresnelReflectance(float3 H, float V, float F0)
{
	float base = 1.0 - dot(V, H);
	float exponential = pow(base, 5.0);
	return exponential + F0 * (1.0 - exponential);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;其中H为标准半角向量。F0为在正入射时的反射值（对于皮肤使用0.028），这个数值来自Beer法则且假定皮肤折射的序号为1.4。
&nbsp;&nbsp;&nbsp;&nbsp;对于BRDF可以进行分解来高效的计算，将BRDF分解为多个预计算的2D纹理。我们可以使用类似的方法来高效计算Kelemen/Szirmay-Kalos镜面反射BRDF，预计算单个纹理（Beckmann分布函数）并使用Schlick的Fresnel近似以得到足够高效的镜面反射计算式，其允许粗糙度在对象中变化。
&nbsp;&nbsp;&nbsp;&nbsp;Beckmann NDF表示为下式:
$$D(h)=\dfrac{e^{-\dfrac{\tan^2\theta_h}{\alpha^2}}}{\alpha^2\cos^4\theta_h}$$
&nbsp;&nbsp;&nbsp;&nbsp;其中$\theta_h$为法线和半角向量的夹角，$\alpha$为粗糙度。
&nbsp;&nbsp;&nbsp;&nbsp;下程序可预计算Beckmann分布纹理，使用指数缩放并对结果值进行二分将函数映射到一定范围来存入一个8位的纹理。其水平轴表示dot(N, H)从0到1的分布，竖向轴表示粗糙度从0到1的分布。
```hlsl
float PHBeckmann(float ndoth, float m)
{
	float alpha = acos(ndoth);
	float ta = tan(alpha);
	float val = 1.0/(m * m * pow(ndoth, 4.0)) * exp(-(ta * ta) / (m * m));
	return val;
}

//渲染一个与屏幕对齐的方框来对一个512×512的纹理进行预计算
float KSTextureCompute(float2 tex : TEXCOORD0)
{
	//缩放结果到[0,1]范围--用于倒置查找
	return 0.5 * pow(PHBeckmann(tex.x, tex.y), 0.1);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;使用上述预计算Beckmann纹理即可得到Kelemen/Szirmay-Kalos镜面模型。
```hlsl
float KS_Skin_Specular(float3 N,//表面法线
					   float3 L,//点到光源的方向
					   float3 V,//点到摄像机的方向
					   float m,//粗糙度
					   float rho_s,//镜面光亮度
					   uniform texobj2D beckmannTex)
{
	float result = 0.0;
	float ndotl = dot(N, L);
	if(ndotl > 0.0)
	{
		float3 h = L + V;//未归一化的半角向量
		float3 H = normalize(h);
		float ndoth = dot(N, H);
		float PH = pow(2.0 * f1tex2D(beckmannTex, float2(ndoth, m)), 10.0);
		float F = fresnelReflectance(H, V, 0.028);
		float frSpec = max(Ph * F / dot(h, h), 0);
		result = ndotl * rho_s * frSpec;
	}
	return result;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;皮肤最外层的组织细胞和体油为电介质材质，他们在反射光时不会对光进行着色。因此，我们对于基于物理的皮肤着色应该使用一个白色的反射颜色。也就是说，白光从皮肤的镜面反射应当白色，有色光的镜面反射应到保持其原来的颜色，无论皮肤的颜色如何。只要所有的渲染在线性颜色空间内渲染完成和正确显示，镜面颜色不应该修改为除白色外的其他颜色。
#### 皮肤的次表面散射
&nbsp;&nbsp;&nbsp;&nbsp;一束光束照射在一个平面上，可以在照射中心处周围看到一个光辉，这是因为光的一部分会传播到表面之下并在周围反射回来。漫反射剖面可以以函数的形式告诉我们到底有多少光显示了出来。对于相同的材质，散射在所有的方向均相同，切与角度无关。每个颜色拥有各自的剖面，红光的散射远快于绿光和蓝光。皮肤的吸收率对于光频率的变化十分敏感。
&nbsp;&nbsp;&nbsp;&nbsp;在确定了合适的漫反射剖面之后，模拟皮肤内次表面散射便简化为收集皮肤上每处的入射光，然后根据剖面的精准信息将其向周围相邻的位置扩散。所有叠加的带颜色的路径汇集起来就会呈现半透明的效果。光在入射后的方向性很快就几乎消失了，因此重要的只是光的总量，之后漫反射光散射到相邻的区域，并沿所有方向平均的流出表面。
&nbsp;&nbsp;&nbsp;&nbsp;对于要模拟的材质，我们需要知道其漫反射剖面的精确形状，偶极或多极漫反射模型基于测得的散射参数对其计算。许多简单的残值可使用较多简单的偶极模型。对于多层构成的材质，每层都有不同的散射属性，就要用多极漫反射模型来计算漫反射剖面。
&nbsp;&nbsp;&nbsp;&nbsp;//高斯和拟合
&nbsp;&nbsp;&nbsp;&nbsp;改进之后的纹理空间漫反射每帧计算步骤如下:
1. 渲染任意阴影贴图
2. 渲染拉伸修正图（可预计算）
3. 渲染辐照度到离屏纹理中
4. 对于每个用于漫反射剖面近似中的高斯内核:
	1. 在U中执行一个可分离模糊渲染过程
	2. 在V中执行一个可分离模糊渲染过程
5. 在3D空间中渲染网格:
	1. 访问每个高斯卷积纹理，并线性组合
	2. 为每个光源加入镜面反射

&nbsp;&nbsp;&nbsp;&nbsp;由于漫反射剖面和6个高斯项的和相适应，因此要计算6个辐照度纹理，对其线性组合（将6个高斯项拟合到皮肤模型漫反射剖面时的线性组合方法）便会生成期望的效果。图像和中的每幅图像由下面的函数卷积得到:
$$I\ast\left(\sum_{i=1}^k\omega_iG(\nu_i,\space r)\right)$$
&nbsp;&nbsp;&nbsp;&nbsp;其中G为高斯项，$r = \sqrt[]{x^2+y^2}$，$I$为辐照度。
&nbsp;&nbsp;&nbsp;&nbsp;由于我们假定材质是均匀的，即散射光在各个方向均相同，因此漫反射剖面是径向均匀的，而高斯项是仅有的既同时可分又径向对称的发散函数。每个高斯卷积纹理可以分开计算且提供的拟合是精确地，这些纹理的带权和可以精确地对原始且不可分函数所得到的较长2D卷积进行近似，这样就可使用可分卷积的和来对不可分卷积做近似。而且两个高斯项的卷积仍然是一个高斯项，这就允许通过对上一个辐照度纹理的结果进行卷积来生成下一个纹理，这样便可通过越来越宽的高斯项对原始图像进行卷积操作，而不会增加每步中tap的数量。两个分别具有变量$\nu_1$和$\nu_2$的径向高斯项按如下方式进行卷积:
$$\begin{split}`  G(\nu_1,\space r) \ast G(\nu_2,\space r) &= \int_0^\infty\int_0^\infty G\left(\nu_1,\space\sqrt[]{x'^2+y'^2} \right) G\left(\nu_2,\space\sqrt[]{(x-x')^2 +(y-y')^2} \right) \,{\rm d}x'\,{\rm d}y' \\
&= G(\nu_1+\nu_2,\space r)\end{split}$$
&nbsp;&nbsp;&nbsp;&nbsp;对一个漫反射剖面使用几个高斯项拟合时，要恒定地把一个宽度大致设置为之前的两倍（方差大约以4的倍数改变），在每步中使用7个tap就相当合适。



















