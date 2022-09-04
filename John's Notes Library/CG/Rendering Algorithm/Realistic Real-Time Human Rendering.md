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
$$\begin{split}G(\nu_1,\space r) \ast G(\nu_2,\space r) &= \int_0^\infty\int_0^\infty G\left(\nu_1,\space\sqrt[]{x'^2+y'^2} \right) G\left(\nu_2,\space\sqrt[]{(x-x')^2 +(y-y')^2} \right) \,{\rm d}x'\,{\rm d}y' \\
&= G(\nu_1+\nu_2,\space r)\end{split}$$
&nbsp;&nbsp;&nbsp;&nbsp;对一个漫反射剖面使用几个高斯项拟合时，要恒定地把一个宽度大致设置为之前的两倍（方差大约以4的倍数改变），在每步中使用7个tap就相当合适。
&nbsp;&nbsp;&nbsp;&nbsp;在皮肤这种弯曲表面上，由于UV畸变，纹理上各位置的距离并不直接对应网格的距离。所以需要处理UV畸变来更精确地处理弯曲表面上的卷积。可以使用如下片段着色器来计算拉伸修正纹理，将头部反扭曲到纹理空间中使用的顶点着色器来计算辐照度纹理，并将纹理的世界空间坐标存储到一个纹理坐标中，屏幕空间中这些坐标的导数能给出拉伸的有效估计，可以将其倒数存储到一个纹理坐标中，用于在卷积着色器中直接对卷积tap的扩散做缩放。
```hlsl
float2 computeStretchMap(float3 worldCoord:TEXCOORD0){
	float3 derivu = ddx(worldCoord);
	float3 derivv = ddy(worldCoord);
	float stretchU = 1.0 / length(derivu);
	float stretchV = 1.0 / length(derivv);
	return float2(stretchU, stretchV);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;一个用于在U方向上对每个纹理进行卷积操作的着色器如下所示。7个tap的最终间隔由拉伸值和将要进行卷积的高斯项宽度共同决定。
```hlsl
float4 convolveU(float2 texCoord:TEXCOORD0,  
   //缩放-用于将高斯项加宽  
   //GaussWidth应当为标准差  
   uniform float GaussWidth,  
   //inputTex-被卷积的纹理  
   uniform texobj2D inputTex,  
   //stretchTex-具有双分量的拉伸纹理  
   uniform texobj2D stretchTex)  
{  
   float scaleConv = 1.0 / 1024.0;  
   float2 stretch = f2tex2D(stretchTex, texCoord);  
   float netFilterWidth = scaleConv * GaussWidth * stretch.x;  
  
   //高斯曲线-标准差为1.0  
   float curve[7] = {0.006,0.061,0.242,0.383,0.242,0.061,0.006};  
  
   float2 coords = texCoord - float2(netFilterWidth * 3.0, 0.0);  
   float4 sum = 0;  
   for(int i=0;i<7;++i)  
   {      
		float4 tap = f4tex2D(inputTex, coords);  
		sum += curve[i] * tap;      
		coords += float2(netFilterWidth, 1.0);  
   } 
	return sum;  
}
```
&nbsp;&nbsp;&nbsp;&nbsp;拉伸映射图对于网格的每个三角形使用常量值，因此将在一些三角形的边上造成不连续性，这对于小的卷积不成问题，但是对于较大的卷积，拉伸值的高频率改变会导致高频率的失真。我们可以复制对辐照度纹理卷积的卷积路径，对拉伸纹理进行卷积，使用拉伸纹理的第k个卷积对第k个辐照度纹理进行卷积。这里的思想是选择一个拉伸映射图，这个拉伸映射图使用卷积操作将要用到的半径拉伸并局部地取均值，再用均值来修正当前辐照度纹理的卷积。在渲染更复杂的、在纹理空间内高度扭曲的表面时，多尺度拉伸修正就会展现出可观的改进效果。更进一步的精确度可以通过尽可能地使用保形映射来得到（或许甚至可以局部地修正卷积方向，使之在纹理空间中变得非正交来保证网格的正交性）。
&nbsp;&nbsp;&nbsp;&nbsp;一个聚焦的白色激光照射到一个平面，一些光亮会被表面所吸收，总的反射量$(r,g,b)$为材质的漫反射颜色（或者说总反射率$R_d$）。对于已知的漫反射剖面可以通过整合所有$r>0$的距离来计算得到，对于每个距离$r$在一个半径为$2\pi r$的圆反射的光亮如下：
$$R_d = \int_0^\infty2\pi rR(r)\,{\rm d}r$$
&nbsp;&nbsp;&nbsp;&nbsp;对于一个用高斯和表达的剖面而言，这很容易计算：$R_d$为高斯和中的权重值（我们选择高斯项定义中的常量因子来得到这个结果）：
$$\int_0^\infty2\pi r\sum_{i=1}^k\omega_iG(\nu_i,\space r)\,{\rm d}r=\sum_{i=1}^k\omega_i$$
&nbsp;&nbsp;&nbsp;&nbsp;偶极和多极理论假定均匀材质沿表面完全一致，因此可以完美预测一个皮肤各处颜色的恒定$R_d$。然而皮肤在颜色方面同时具有高频和低频的变化，这使得要实时对皮肤组织层间散射和吸收特性的变化模拟是不切实际的，因此我们可以通过使用纹理来对这个复杂的过程进行效果模拟。我们可以在下面几种将漫反射颜色图整合到散射模型的方法中选择一个，由于模型来自对完全一致材质的假定，因此这些技术均不是完全基于物理的，并且每种技术在不同的场景中可能会有各自的益处。
&nbsp;&nbsp;&nbsp;&nbsp;后置散射变形方法是首先不进行任何着色地执行所有散射计算，生成一个各处被平均成白色的值，然后与漫反射颜色图相乘得到希望的肤色。漫反射颜色纹理在散射后进行乘法运算，因此所有的高频纹理细节被保留下来，这是因为它没有被卷积路径所模糊，并没有发生肤色中颜色的扩散，这使得此方法存在缺陷。
&nbsp;&nbsp;&nbsp;&nbsp;另一个引入颜色变化的方法是预散射变形，在计算辐照度纹理时，先相乘光照与漫反射颜色，然后并不在最终路径中使用。这样漫反射颜色的散射和光照的散射相同，是后者模糊且柔和。但是缺点就是这种方法会丢失过多的高频细节，如胡茬等。我们可以很容易的将两种技术结合到一起，可以提供一定的颜色扩散，同时也能在颜色图中还保持一定的高频细节。
&nbsp;&nbsp;&nbsp;&nbsp;我们可以将一部分漫反射颜色应用到预散射中，剩余部分应用到后置散射中。可以在模糊之前通过``pow(diffuseColor,mix)``与光线相乘，并通过``pow(diffuseColor,1-mix)``来乘以所有模糊过纹理的最终整合结果。mix值决定了预散射和后置散射各自应用于多少着色。一个0.5的mix值对应于皮肤表面最上层的一个无穷小吸收层，它精确地对光进行两次吸收：一次在进入皮肤但未散射前，另一次是在离开时。当mix值为0.5时，可以用``sqrt(diffuseColor)``替代``pow(diffuseColor,0.5)``来获得更好的性能。
&nbsp;&nbsp;&nbsp;&nbsp;一个对次表面散射的近似是使用不用的法线贴图来计算镜面/漫反射光照比，来模拟将漫反射光照变得柔和。镜面反射光直接反射出表面，因此它反映出皮肤表面最剧烈高频的变化。余下的光照是穿过皮肤的漫反射，由于表面的细节使光照的高频变化变柔和，这些表面细节可能为凹凸变化、孔隙、褶皱、皱纹等。而红色光在皮肤的次表面散射的最远，柔化效果也就最强。直接将法线贴图进行模糊并不能正确得到肤色或者是实际光照本身的漫反射效果，这是因为之前提供的方法已经直接对次表面散射进行了建模，这影响了表面细节的柔化，实际的法线信息应该同时被应用于镜面反射和漫反射的计算中。
&nbsp;&nbsp;&nbsp;&nbsp;下面程序就是最终渲染路径中的片段着色器。
```hlsl
float4 finalSkinShader(float3 position:POSITION,  
   float2 texCoord : TEXCOORD0,  
   float3 normal:TEXCOORD1,  
  
   //用于修改后半透明阴影贴图的阴影贴图坐标  
   float4 TSM_coord:TEXCOORD2,  
  
   //模糊后的辐照度纹理  
   uniform texobj2D irrad1Tex,  
   uniform texobj2D irrad2Tex,  
   uniform texobj2D irrad3Tex,  
   uniform texobj2D irrad4Tex,  
   uniform texobj2D irrad5Tex,  
   uniform texobj2D irrad6Tex,  
  
   //定义皮肤剖面的RGB高斯权重  
   uniform float3 gauss1w,  
   uniform float3 gauss2w,  
   uniform float3 gauss3w,  
   uniform float3 gauss4w,  
   uniform float3 gauss5w,  
   uniform float3 gauss6w,  
  
   //预散射与后置散射比值  
   uniform float mix,  
  
   uniform texobj2D TSMTex,  
   uniform texobj2D rhodTex  
   ){  
   //离开表面的总散射光量  
   float3 diffuseLight = 0;  
  
   float4 irrad1tap = f4tex2D(irrad1Tex, texCoord);  
   float4 irrad2tap = f4tex2D(irrad2Tex, texCoord);  
   float4 irrad3tap = f4tex2D(irrad3Tex, texCoord);  
   float4 irrad4tap = f4tex2D(irrad4Tex, texCoord);  
   float4 irrad5tap = f4tex2D(irrad5Tex, texCoord);  
   float4 irrad6tap = f4tex2D(irrad6Tex, texCoord);  
  
   diffuseLight += gauss1w * irrad1tap.xyz;  
   diffuseLight += gauss2w * irrad2tap.xyz;  
   diffuseLight += gauss3w * irrad3tap.xyz;  
   diffuseLight += gauss4w * irrad4tap.xyz;  
   diffuseLight += gauss5w * irrad5tap.xyz;  
   diffuseLight += gauss6w * irrad6tap.xyz;  
  
   //重新将漫反射剖面归一化到白色  
   float3 normConst = gauss1w + gauss2w + gauss3w + gauss4w + 
	   gauss5w + gauss6w;  
   diffuseLight /= normConst;//重新归一化到白色漫反射光  
  
   //从修改过的TSM计算全局散射  
   //TSMtap = (distance to light, u, v)  
   float3 TSMtap = f3tex2D(TSMTex, TSM_coord.xy / TSM_coord.w);  
   //从透过对象的四个平均厚度(以mm为单位)  
   float4 thickness_mm = 1.0 * -(1.0/0.2) * 
	   log(float4(irrad2tap.w, irrad3tap.w,irrad4tap.w,irrad5tap.w));  
   float2 stretchTap = f2tex2D(stretch32Tex, texCoord);  
   float stretchval = 0.5 * (stretchTap.x + stretchTap.y);  
   float4 a_values = float4(0.433,0.753,1.412,2.722);  
   float4 inv_a = -1.0/(2.0 * a_values * a_values);  
   float fades = exp(thickness_mm * thickness_mm * inv_a);  
  
   float textureScale = 1024.0 * 0.1 / stretchval;  
   float blendFactor4 = saturate(textureScale * 
	   length(v2f.c_texCoord.xy - TSMtap.yz)/(a_values.y*6.0));  
   float blendFactor5 = saturate(textureScale * 
	   length(v2f.c_texCoord.xy - TSMtap.yz)/(a_values.z*6.0));  
   float blendFactor6 = saturate(textureScale * 
	   length(v2f.c_texCoord.xy - TSMtap.yz)/(a_values.w*6.0));  
  
   diffuseLight += gauss4w / normConst * fades.y * 
	   blendFactor4 * f3tex2D(irrad4Tex,TSMtap.yz).xyz;  
   diffuseLight += gauss5w / normConst * fades.y * 
	   blendFactor5 * f3tex2D(irrad5Tex,TSMtap.yz).xyz;  
   diffuseLight += gauss6w / normConst * fades.y * 
	   blendFactor6 * f3tex2D(irrad6Tex,TSMtap.yz).xyz;  
  
   //从一个diffuseColor颜色图决定皮肤颜色  
   diffuseLight *= pow(f3tex2D(diffuseColorTex, texCoord), 1.0-mix);  
  
   //能量保存（可选）-roh_s和m可以绘制在一个纹理中  
   float finalScale = 1 - rho_s * f1tex2D(rhodTex, float2(dot(N,V), m));  
   diffuseLight *= finalScale;  
  
   float3 specularLight = 0;  
   //为每个光照计算镜面反射  
   for(each light)  
      specularLight += lightColor[i] * lightShadow[i] * 
	      KS_Skin_Specular(N, L[i], V, m, rho_s, beckmannTex);  
   return float4(diffuseLight + specularLight, 1.0);  
}
```








