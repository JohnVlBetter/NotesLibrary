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
&nbsp;&nbsp;&nbsp;&nbsp;纹理缝隙会对纹理空间散射带来麻烦，因为在网格上连接的区域在纹理空间上并不相连，不能够容易的在它们之间漫反射。纹理空间上的空区域将沿每个缝隙边模糊到网格上，在渲染阶段，将卷积后的辐照度纹理应用回网格上时，就造成了失真。如果使用的UV映射充满了整个纹理空间，那基本看不出难以接受的结果。
&nbsp;&nbsp;&nbsp;&nbsp;在渲染中我们需要保持精确的能量守恒，由于我们渲染的是皮肤，因此我们只讨论次表面散射的情况。次表面散射的所有可用能量精确地等于所有未被镜面BRDF所直接反射的能量。一次镜面反射计算给出总的输入能量从单个方向反射出的比例。要计算镜面反射所使用的的总能量，所有的出射方向都必须考虑到。因此在将光照存储到辐照度纹理中之前，需要沿半球所有方向整合镜面反射量，并将漫反射光亮与剩余的能量相乘。若$f_r$为镜面BRDF，L为位于表面$x$位置的光向量且$\omega_o$为半球中关于N的向量，那么就可以用下面的式子为每个光照减少漫反射光
$$\rho_{dt}(x,L) = 1-\int_{2\pi}f_r(x,\omega_o,L)(\omega_o\cdot N)\,{\rm d}\omega_o$$
&nbsp;&nbsp;&nbsp;&nbsp;使用球面坐标，上式的积分计算方法为
$$\int_0^{2\pi}\int_0^{\pi/2}f_r(x,\omega_o,L)(\omega_o\cdot N)sin(\theta)\,{\rm d}\theta\,{\rm d}\phi$$
&nbsp;&nbsp;&nbsp;&nbsp;这个值会根据位置$x$处的粗糙度$m$，及$N\cdot L$而改变。我们从镜面BRDF中移除$\rho_s$常数，并预计算式2中的积分。由于一致采样的缘故，这仅仅是对2D积分的粗糙估计。一致采样积分对于精确度较高的值而言效果尤其差，但这样的数值在我们的皮肤着色器中并没有用到。预计算可以在下述程序中执行得出。
```hlsl
float computeRhodtTex(float2 tex : TEXCOORD0)  
{  
	//在半球表面对镜面反射BRDF分量进行积分  
	float costheta = tex.x;  
	float pi = 3.14159265358979324;  
	float m = tex.y;  
	float sum = 0.0;  
	//更多的项可用来得到更高的精确度  
	int numterms = 80;  
	float3 N = float3(0.0,0.0,1.0);  
	float3 V = float3(0.0,sqrt(1.0 - costheta * costheta), costheta);  
	for(int i = 0; i < numterms; ++i)  
	{      
		float phip = float(i) / float(numterms - 1) * 2.0 * pi;  
		float localsum = 0.0f;  
		float cosp = cos(phip);  
		float sinp = sin(phip);  
		for(int j = 0; j < numterms; ++j)  
		{         
			float thetap = float(j) / float(numterms - 1) * pi * 0.5;  
			float sint = sin(thetap);  
			float cost = cos(thetap);  
			float3 L = float3(sinp * sint, cosp * sint, cost);  
			localsum += KS_Skin_Specular(V, N, L, m, 0.0277778)
				 * sint;  
		}      
		sum += localsum * (pi * 0.5) / float(numterms);  
	}   
	return sum * (pi * 2.0) / (float(numterms));  
}
```
&nbsp;&nbsp;&nbsp;&nbsp;我们可以通过使用这个$\rho_{dt}$纹理来修改漫反射光照计算。在产生辐照度纹理时，对每个光源执行一次。环境光照并没有L方向，在技术上应该乘以一个基于$\rho_{dt}$半球积分的常数，但由于这仅为一个常数（除了基于$m$的变量）并且很可能接近于1.0，因此不必进行计算。正确的结果可以通过使用$\rho_{dt}$的方向依赖性对原始立方体贴图进行卷积来替代仅仅使用$N\cdot L$而得到。下程序可使用$\rho_{dt}$纹理来计算一个点光源的漫反射辐照度。
```hlsl
float ndotL[i] = dot(N, L[i]);  
float3 finalLightColor[i] = LColor[i] * LShadow[i];  
float3 Ldiffsue[i] = saturate(ndotL[i]) * finalLightColor[i];  
float3 rho_dt_L[i] = 1.0 - rho_s * 
	f1tex2D(rhodTex, float2(ndotL[i], m));  
irradiance += Ldiffsue[i] * rho_dt_L[i];
```
&nbsp;&nbsp;&nbsp;&nbsp;光照在表面之下发散之后，它必须穿过由BRDF支配的同样的粗糙接触面，并且方向效果在此再次生效。基于我们从表面观察的方向(基于$N\cdot V$)，一个不同的漫反射光照量将逃逸。由于我们使用了对漫反射的近似，因此可以将逃逸的光照看作等量地沿着各方向流出，如它们到达表面时一样。因此另一个积分项必须考虑一个由入射方向组成的半球，以及一个射出的V方向。由于基于物理的BRDF可逆，我们就可以重复使用同一个$\rho_{dt}$项，但这次索引基于V，而非L，如最终渲染路径的片段着色器代码的最后所展示的。
&nbsp;&nbsp;&nbsp;&nbsp;能量保持的一个不足之处就是没有考虑到互反射，导致模型丢失了应当在颧骨区域中不断反弹的光亮，这些光亮会照亮鼻子的边缘。使用预计算的辐照度传递可以捕捉这样的互反射，但是该技术不支持变动的或者可变的模型，因此不能应用于人体。
&nbsp;&nbsp;&nbsp;&nbsp;纹理空间漫反射不会完全捕捉光透过薄区域（如耳朵、鼻翼等）的传播情况。在这些薄区域中，两个表面的三维空间可能很接近，但是在纹理空间中却可能相隔很远。我们可使用一种方法对半透明阴影贴图（translucent shadow maps）进行修改，重用卷积后的辐照度纹理对穿过薄区域的漫反射进行高效地估计。
&nbsp;&nbsp;&nbsp;&nbsp;与传统半透明阴影贴图不同的是，我们渲染向光表面的$z$和$(u,v)$坐标这允许在阴影中的每点计算透过对象的厚度，并在表面另一侧访问卷积后的辐照度纹理。我们在一个三分量的阴影贴图中存储向光表面的$z$和$(u,v)$坐标，这使得阴影中的区域可以计算透过表面的深度，并访问另一侧光照的模糊版本。
![阴影内区域C计算在位置B传输的光线](/assets/img/CG/RealisticReal-TimeHumanRendering/阴影内区域C计算在位置B传输的光线.jpg "阴影内区域C计算在位置B传输的光线")
&nbsp;&nbsp;&nbsp;&nbsp;如上图所示，对于任意处于阴影中的位置$C$，TSM提供了向光表面上点$A$的距离$m$和UV坐标。我们希望对$C$处射出的散射光进行估计，其为剖面$R$上向光点的卷积，其中从距离$C$到每个样本的距离分别单独计算。对$B$进行计算式，实际上容易很多，对于较小的角度$\theta$，$B$将十分接近于$C$。对于较大的$\theta$，Fresnel和$N\cdot L$项将使得影响效果很小，并隐藏了大部分错误，因此很可能会并不值得计算$C$点的正确数值。计算$B$处的散射，向光表面上距$A$点$r$距离的任意样本需要使用下面的卷积：
$$\begin{split}R\left(\sqrt{r^2+d^2}\right) &=
\sum_{i=1}^k\omega_iG\left(\nu_i,\sqrt{r^2+d^2}\right) \\ &=
\sum_{i=1}^k\omega_ie^{-d^2/v_i}G(\nu_i,r)\end{split}$$
&nbsp;&nbsp;&nbsp;&nbsp;之前已经利用过三维空间中高斯项的可分性将关于$\sqrt{r^2+d^2}$的函数变换到关于$r$的函数。由于向光点已经在$A$处由$G(\nu_i,r)$进行了卷积，这一步十分方便。因此在位置$C$（使用$B$来代替进行估算）处射出的总漫反射光量为$k$次纹理查找（对存储在TSM中位置$A$使用$(u,v)$）的和，每次纹理查找的权重由$\omega_i$和一个基于$\nu_i$和$d$的指数项决定。由于每个高斯项在高斯和漫反射剖面可分，因此这种变换是有效的。
&nbsp;&nbsp;&nbsp;&nbsp;在阴影贴图计算得到的深度值$m$被$cos(\theta)$修正，这是因为使用了对漫反射的近似。通过对比$A$和$C$处的表面法线，这种修正只会在发现朝向相反时用到。当两个法线相互垂直时，使用一个lerp操作将混合修正回原始的距离$m$。虽然设定是一个平整的平面，但是应用到非平整的平面上效果也相当好。
&nbsp;&nbsp;&nbsp;&nbsp;目前的方式会有一些问题，因为我们只考虑了光透过对象的一条路径。由TSM计算得到的深度中的高频变化可能导致输出中的高频失真，这不应该出现在高度散射的半透明效果中，因为半透明效果模糊了所有的传播光，所以不应该出现尖锐的效果。我们可以通过将透过表面的深度存储在辐照度纹理的alpha通道中，其随着辐照度纹理在表面被模糊。在计算全局散射时，每个高斯项使用对应深度的模糊版本。而要将距离值压缩到一个8位的通道中，可以将常数$d$存储为exp(-const * d)，沿表面对深度的这个指数进行卷积，并在查找时对其取反。程序如下图所示：
```hlsl
float4 computeIrradianceTexture()
{   
	float distanceToLight = length(worldPointLight0Pos – worldCoord);   
	float4 TSMTap = f4tex2D(TSMTex, TSMCoord.xy / TSMCoord.w);
	// Find normal on back side of object, Ni
	float3 Ni = f3tex2D(normalTex, TSMTap.yz) * float3(2.0, 2.0, 2.0) 
		- float3(1.0, 1.0, 1.0);
	float backFacingEst = saturate(-dot(Ni, N));   
	float thicknessToLight = distanceToLight - TSMTap.x;   
	// Set a large distance for surface points facing the light    
	if( ndotL1 > 0.0 )   
	{     
		thicknessToLight = 50.0;   
	}   
	float correctedThickness = saturate(-ndotL1) * thicknessToLight;
	float finalThickness = lerp(thicknessToLight, correctedThickness,
		backFacingEst);
	// Exponentiate thickness value for storage as 8-bit alpha    
	float alpha = exp(finalThickness * -20.0);   
	return float4(irradiance, alpha); 
}
```
&nbsp;&nbsp;&nbsp;&nbsp;对于多光源和环境光照，我们理应为每个光照生成卷积后的辐照度纹理并分开存储，但是这会消耗更多的纹理内存。对此我们可以计算所有透过表面深度的最小值，并沿表面对此积分。这虽然只是一个近似，但在多光源情况下，算法消耗不会随光源数目而成倍增加。
&nbsp;&nbsp;&nbsp;&nbsp;用于对子表面散射卷积进行加速的技术同样适用于其他不可分的卷积，如布隆过滤器。Pharr and Humphreys 2004 推荐了一个$(1-r/d)^4$的布隆过滤器，来向最终渲染的图像增加光晕效果。虽然这个过滤器是不可分的，但可以通过两个高斯项的和对其进行出色的模拟，并且同样的用于改进纹理空间漫反射的可分、层次化卷积技术。对于d=1，$(1-r^4)$可通过$0.0174G(0.008,r)+0.192G(0.0576,r)$来很好地近似。
#### 结论
&nbsp;&nbsp;&nbsp;&nbsp;新算法基于纹理空间漫反射和半透明阴影贴图生成了高真实感的人皮肤图像。对于大部分角色使用至多6个高斯项对漫反射剖面进行模拟在外观上就已足够。可以使用一个细节等级系统来将多个漫反射精度在不同距离进行切换，来更好的避免明显的效果跳变。最终的渲染效果如下图所示：
![最终结果](/assets/img/CG/RealisticReal-TimeHumanRendering/result.jpg "最终结果")













