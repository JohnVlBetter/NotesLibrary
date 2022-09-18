# 高斯模糊
&nbsp;&nbsp;&nbsp;&nbsp;高斯模糊也叫高斯平滑，是一种高品质的模糊算法，可以用于减少图像噪声及降低细节层次，通常对图像进行模糊以实现特殊的后处理效果。高斯模糊其实就是利用卷积滤波来过滤掉图像的高频信号，保留下图像的低频信息。
&nbsp;&nbsp;&nbsp;&nbsp;在二维空间中，高斯模糊方程可以表示为：
$$G(u,v) = \dfrac{1}{2\pi\sigma^2}e^{-\dfrac{(u^2+v^2)}{2\sigma^2}}$$
&nbsp;&nbsp;&nbsp;&nbsp;其中模糊半径$r$为$\sqrt{(u^2+v^2)}$。
&nbsp;&nbsp;&nbsp;&nbsp;由于高斯模糊满足线性可分，也就是可以将二维空间中图像的模糊计算拆分为两个单独的一维空间分别进行计算。在计算量的角度来说，这样的拆分大大降低了算法的时间复杂度。高斯核的线性分解证明如下：
$$Kernel(i,j)=\dfrac{1}{2\pi\sigma^2}e^{-\dfrac{(u^2+v^2)}{2\sigma^2}}$$
$$G(i,j) = \dfrac{1}{2\pi\sigma^2}\sum_{m=1}\sum_{n=1}e^{-\dfrac{(u^2+v^2)}{2\sigma^2}}f(i-m,j-n)$$
$$G(i,j) = \dfrac{1}{2\pi\sigma^2}\sum_{m=1}e^{-\dfrac{u^2}{2\sigma^2}}\sum_{n=1}e^{-\dfrac{v^2}{2\sigma^2}}f(i-m,j-n)$$
&nbsp;&nbsp;&nbsp;&nbsp;由此可得出高斯模糊对应Shader的实现：
```hlsl
Shader "PostEffect/GaussianBlur"
{
    Properties
    {
        _MainTex("Main Texture", 2D) = "white" { }
    }

    SubShader
    {

        Cull Off
        ZWrite Off
        ZTest Always
        Tags{
            "RanderPipline" = "UniversalPipeline"
        }

        HLSLINCLUDE

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _MainTex_ST;
                float _Radius;
            CBUFFER_END
  
            TEXTURE2D (_MainTex);
            SAMPLER(sampler_MainTex);
  
            struct appdata
            {
                float4 vertex: POSITION;
                float2 uv: TEXCOORD0;
            };

            struct v2f
            {
                float2 uv: TEXCOORD0;
                float4 vertex: SV_POSITION;
            };

        ENDHLSL

        Pass
        {
            Name "ForwardUnlit"
            Tags {"LightMode" = "UniversalForward"}
  
            HLSLPROGRAM
                #pragma vertex vert
                #pragma fragment frag
                
                v2f vert(appdata v)
                {
                    v2f o;
                    o.vertex = TransformObjectToHClip(v.vertex);
                    o.uv = v.uv;
                    return o;
                }

                float4 frag(v2f i) : SV_Target
                {
                    half4 c = 0;
                    
                    float2 uv = i.uv;
                    half2 offset = _Radius * half2(1, 0);

                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * -2.0) * 0.0545;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * -1.0) * 0.2442;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex,
	                    uv + offset * 0.0) * 0.4026;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * 1.0) * 0.2442;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * 2.0) * 0.0545;
  
                    return c;
                }
            ENDHLSL
        }

        Pass
        {
            Name "SRPDefaultUnlit"
            Tags {"LightMode" = "SRPDefaultUnlit"}

            HLSLPROGRAM
  
                #pragma vertex vert
                #pragma fragment frag

                v2f vert(appdata v)
                {
                    v2f o;
                    o.vertex = TransformObjectToHClip(v.vertex);
                    o.uv = v.uv;
                    return o;
                }

                float4 frag(v2f i) : SV_Target
                {
                    half4 c = 0;

                    float2 uv = i.uv;
                    half2 offset = _Radius * half2(0, 1);
  
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * -2.0) * 0.0545;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * -1.0) * 0.2442;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * 0.0) * 0.4026;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * 1.0) * 0.2442;
                    c += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, 
	                    uv + offset * 2.0) * 0.0545;
  
                    return c;
                }
            ENDHLSL
        }
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;对应在Unity URP中的RenderFeature及RenderPass类实现如下：
```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class GaussianBlurRenderFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings {
        public Shader gaussianShader;
    }

    GaussianBlurPass gaussianBlurPass;
    public Settings settings;

    public override void Create()
    {
        gaussianBlurPass = new GaussianBlurPass(RenderPassEvent.BeforeRenderingPostProcessing, settings.gaussianShader);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        gaussianBlurPass.SetupSrc(renderer.cameraColorTarget);
        renderer.EnqueuePass(gaussianBlurPass);
    }
}

public class GaussianBlurPass : ScriptableRenderPass
{
    static readonly int radius = Shader.PropertyToID("_Radius");
    static readonly int dst = Shader.PropertyToID("_GaussianBlurTarget");

    static readonly string cmdName = "GaussianBlurEffect";
    GaussianBlurVolume gaussianBlurVolume;
    Material gaussianBlurMaterial;
    RenderTargetIdentifier src;
    CommandBuffer cmd;

    public void SetupSrc(in RenderTargetIdentifier src){
        this.src = src;
    }

    public GaussianBlurPass(RenderPassEvent evt, Shader gaussianShader){
        renderPassEvent = evt;
        if (gaussianShader == null){
            Debug.LogError("Gaussian blur shader not found.");
            return;
        }
        gaussianBlurMaterial = CoreUtils.CreateEngineMaterial(gaussianShader);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData){
        if (!renderingData.cameraData.postProcessEnabled) 
            return;

        if (gaussianBlurMaterial == null){
            Debug.LogError("Gaussian blur material not created.");
            return;
        }

        gaussianBlurVolume = VolumeManager.instance.stack.GetComponent<GaussianBlurVolume>();
        if (gaussianBlurVolume == null) { return; }
        if (!gaussianBlurVolume.IsActive()) { return; }

        cmd = CommandBufferPool.Get(cmdName);

        var width = (int)(renderingData.cameraData.camera.scaledPixelWidth / gaussianBlurVolume.downSample.value);
        var height = (int)(renderingData.cameraData.camera.scaledPixelHeight / gaussianBlurVolume.downSample.value);
        gaussianBlurMaterial.SetFloat(radius, gaussianBlurVolume.BlurRadius.value * .0001f);

        int shaderPass = 0;
        cmd.SetGlobalTexture(Shader.PropertyToID("_MainTex"), src);
        cmd.GetTemporaryRT(dst, width, height, 0);

        cmd.Blit(src, dst);
        for (int i = 0; i < gaussianBlurVolume.Iteration.value; i++)
        {
            cmd.GetTemporaryRT(dst, width / 2, height / 2, 0);
            cmd.Blit(dst, src, gaussianBlurMaterial, shaderPass);
            cmd.Blit(src, dst);
            cmd.Blit(dst, src, gaussianBlurMaterial, shaderPass + 1);
            cmd.Blit(src, dst);
        }
        for (int i = 0; i < gaussianBlurVolume.Iteration.value; i++)
        {
            cmd.GetTemporaryRT(dst, width * 2, height * 2, 0);
            cmd.Blit(dst, src, gaussianBlurMaterial, shaderPass);
            cmd.Blit(src, dst);
            cmd.Blit(dst, src, gaussianBlurMaterial, shaderPass + 1);
            cmd.Blit(src, dst);
        }

        cmd.Blit(dst, dst, gaussianBlurMaterial, 0);
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;为了方便控制高斯模糊的各个可变值，实现了如下的Volume类：
```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class GaussianBlurVolume : VolumeComponent, IPostProcessComponent
{
    public bool IsActive() => BlurRadius.value > 0f;
    public bool IsTileCompatible() => false;

    [Range(0f, 1000f), Tooltip("模糊范围")]
    public FloatParameter BlurRadius = new FloatParameter(10f);

    [Range(0, 10), Tooltip("迭代次数")]
    public IntParameter Iteration = new IntParameter(3);

    [Range(1, 10), Tooltip("降采样")]
    public FloatParameter downSample = new FloatParameter(1f);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;最终在Unity中的实现效果如下图所示：
![未模糊](/assets/img/CG/GaussianBlur/dragon.jpeg "未模糊")
![高斯模糊](/assets/img/CG/GaussianBlur/dragon_gaussianBlur.jpeg "高斯模糊")