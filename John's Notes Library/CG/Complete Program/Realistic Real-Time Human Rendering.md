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