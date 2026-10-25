---
title: "视差 Shader：从原理到 URP 实现"
date: 2026-10-24
draft: false
math: true
layout: single
ShowToc: true
tags:
  - Unity
  - Shader
  - URP
  - POM
categories:
  - 图形学
description: >-
  从视差贴图基础原理出发，逐步解析 Unity URP 中的视差 Shader、视差遮蔽映射、场景光照与高度图自阴影实现。
---
本文从零开始，解释 Unity Shader 怎样读取网格、观察方向和贴图，并用它们实现基础视差贴图与视差遮蔽映射（Parallax Occlusion Mapping，POM）。示例项目使用 Unity 2022.1.25f1 和 URP 13.1.9。

## 1. 先认识 Shader 在做什么

网格（Mesh）由三角形组成。网格顶点保存位置、法线、切线和 UV 等数据。UV 是贴图上的二维坐标：通常左下角是 `(0, 0)`，右上角是 `(1, 1)`。

Unity 的渲染管线（Render Pipeline）负责安排一帧画面的工作。URP 是 Unity 提供的一条可配置渲染管线。Shader（着色器）描述某次绘制怎样处理输入并产生输出；Material（材质）则把一个 Shader 和具体的颜色、贴图、数值绑定起来。

一个常见的 HLSL Shader 有两个阶段：

1. **顶点阶段（Vertex）**：对网格顶点运行。它把顶点位置变换到屏幕使用的坐标，并把片元阶段需要的数据传过去。
2. **片元阶段（Fragment）**：对网格覆盖的像素运行。它采样纹理、计算颜色，并将结果输出给当前渲染 Pass。

Unity 的 Shader 文件还用 **ShaderLab** 描述材质面板属性、哪些管线能使用 Shader，以及每个绘制 Pass 的状态。ShaderLab 外壳中间的 `HLSLPROGRAM` 才是 HLSL 程序。

## 2. 视差贴图的想法

普通纹理只根据网格 UV 找颜色。观察者移动时，表面上的小纹理不会相对移动，因此看起来像贴在平面上。

视差贴图（Parallax Mapping）用一张高度图描述纹理表面的高低，再根据观察方向移动 UV。它改变的是**从哪里读取颜色**，而不是网格顶点的位置。

### 2.1 符号与假设

以下量都是二维或三维向量；小写 `h` 是一个标量：

- `u = (u, v)`：网格当前的二维 UV 坐标。
- `h(u) ∈ [0, 1]`：高度图在 `u` 处的灰度。项目约定白色较高、黑色较低。
- `V = (Vx, Vy, Vz)`：从表面指向相机的观察方向，已变换到切线空间并归一化。切线空间以表面切线、副切线和法线作为三个坐标轴，因此 `Vz` 表示观察方向朝向表面法线的分量。
- `s`：材质中的视差强度，单位是 UV；越大，纹理偏移越明显。
- `c = h - 0.5`：将高度以中间灰度为零点，使偏移可正可负。
- `u'`：偏移后的 UV，用于读取颜色图。

观察方向越贴近表面，`Vz` 越接近零；直接除以这样的数会产生很大的偏移。因此示例用 `max(Vz, 0.08)` 限制分母。

### 2.2 基础视差：一次采样、一次偏移

代码先读取当前 UV 的高度，再用观察方向计算偏移：

```text
h = HeightMap(u).r
c = h - 0.5
Vsafe.z = max(Vz, 0.08)
Δu = (Vx, Vy) / Vsafe.z × c × s
u' = u - Δu
color = BaseMap(u') × BaseColor
```

这里 `.r` 表示读取高度纹理的红色通道。灰度图的 RGB 通常相同，所以用红色通道即可。高度为中灰时 `c = 0`，UV 不偏移；高度较高或较低时偏移方向相反。越斜着看，`(Vx, Vy)/Vz` 的长度越大，视差越明显。

这是一种快速近似：它只在原 UV 位置采样一次高度，没有沿视线寻找真正的表面交点，也没有考虑细节之间互相遮挡。网格轮廓、深度缓冲和阴影也不会因此改变。

### 2.3 POM：沿视线寻找高度图交点

基础视差将高度直接换算成一个偏移。POM（视差遮蔽映射）则在高度图上沿视线前进，一步步寻找观察射线与高度表面的交点。交点的 UV 用来读取颜色图，因此前景的高处可以遮住后方的低处。

为方便描述，定义：

- `t ∈ [0, 1]`：沿虚拟高度范围前进的比例，`0` 在高度范围上方，`1` 在范围底部。
- `D(u) = 1 - h(u)`：高度图对应的归一化深度，白色高度映射到较浅处，黑色高度映射到较深处。
- `d = (Vx, Vy) / max(Vz, 0.08) × s`：走完整个虚拟高度范围时，射线在 UV 平面上的总位移。
- `u(t) = u - d × t`：射线在步进比例 `t` 时读取高度图的位置。
- `F(t) = t - D(u(t))`：射线深度与高度图深度的差。首次从负值变为非负值的位置，就是射线穿过高度表面的区间。

算法流程：

```text
根据观察方向选择步数 N（斜视时多走几步）
prevT = 0
prevF = F(0)
对 i 从 1 到 N：
    t = i / N
    当前射线 UV = u - d × t
    当前差值 F = t - (1 - 高度图(当前射线 UV))
    若 prevF < 0 且当前 F >= 0：
        在上一个点和当前点之间插值，估计交点
        返回交点 UV
    prevT = t，prevF = 当前 F
返回最后一个射线 UV
```

相邻采样点只把交点夹在一个小区间内。实现再用两个差值按线性比例插值，细化交点，避免 UV 只停在较粗的步进格点上。增加步数通常会让交点更准确，但也会增加片元着色器的高度图采样次数，降低性能。

POM 能表现贴图内部的遮挡和更可信的视角变化，但它仍然在原网格的三角形上运行。除非额外写入匹配的深度、法线与阴影数据，它不会改变真实几何轮廓、相机深度或投射到其他物体上的阴影。

## 3. 先看 Unity Shader 的文件结构

目标文件：`Assets/RenderingLearning/Shaders/HLSL/04_CodeParallaxUnlit.shader`。这里按代码顺序解释关键语句；空行和单独的花括号只用于分隔代码块。

### 3.1 Shader 声明与材质属性

```hlsl
Shader "Learning/Code Parallax Unlit"
```

这声明 Shader 在 Unity 材质面板中的名字。它不是磁盘文件名；斜线会让 Unity 把它显示为分组路径。

```hlsl
Properties
{
    [MainTexture] _BaseMap ("Base Map", 2D) = "white" {}
    [MainColor] _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
    _HeightMap ("Height Map (White = High)", 2D) = "gray" {}
    _ParallaxScale ("Parallax Scale", Range(0, 0.1)) = 0.035
}
```

- `Properties` 中的项目会显示在材质 Inspector 中。每个属性由 HLSL 中同名变量读取。
- `[MainTexture]` 和 `[MainColor]` 告诉 Unity 哪个属性是主纹理和主颜色。
- `_BaseMap` 是颜色图；`2D` 表示二维纹理，缺省值为白色。
- `_BaseColor` 是乘到纹理上的 RGBA 颜色，缺省为白色（不改变纹理）。
- `_HeightMap` 是高度图；项目把白色解释为高处，缺省中灰表示没有明显高低差。
- `_ParallaxScale` 是材质中的强度滑条，缺省 `0.035`，Inspector 范围为 `0` 到 `0.1`。

### 3.2 SubShader、Pass 与渲染状态

```hlsl
SubShader
{
    Tags { "RenderPipeline"="UniversalPipeline" "RenderType"="Opaque" "Queue"="Geometry" }
```

- `SubShader` 包含一组供 Unity 选择的 Pass。
- `RenderPipeline` 声明该 SubShader 面向 URP；其他管线不一定能执行这里的 URP HLSL。
- `RenderType="Opaque"` 标记它属于不透明物体。
- `Queue="Geometry"` 将它放进不透明几何体的绘制队列。

```hlsl
Pass
{
    Name "ForwardUnlit"
    Tags { "LightMode"="UniversalForwardOnly" }
    Blend One Zero
    ZWrite On
    ZTest LEqual
    Cull Back
```

- `Pass` 描述一次绘制。当前颜色 Pass 的名字是 `ForwardUnlit`。
- `LightMode="UniversalForwardOnly"` 让 URP 在前向绘制中使用这个 Pass。
- `Blend One Zero` 表示输出颜色直接覆盖目标颜色，不与原颜色混合。
- `ZWrite On` 将片元深度写入深度缓冲，供之后的物体进行遮挡判断。
- `ZTest LEqual` 只通过“深度小于或等于已有深度”的测试。
- `Cull Back` 不绘制背向观察者的三角形背面。

### 3.3 HLSL 入口、贴图和材质常量

```hlsl
HLSLPROGRAM
#pragma vertex Vert
#pragma fragment Frag
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
```

- `HLSLPROGRAM` 到 `ENDHLSL` 之间是这一 Pass 的 HLSL 程序。
- `#pragma vertex Vert` 指定顶点阶段入口函数为 `Vert`。
- `#pragma fragment Frag` 指定片元阶段入口函数为 `Frag`。
- `Core.hlsl` 提供 URP 的坐标变换、顶点输入、纹理采样等常用函数。

```hlsl
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);
TEXTURE2D(_HeightMap);
SAMPLER(sampler_HeightMap);
```

`TEXTURE2D` 声明二维纹理，`SAMPLER` 声明其采样器。采样器包含过滤和寻址等采样设置。后面的 `SAMPLE_TEXTURE2D` 会用这两者从纹理读取一个颜色。

```hlsl
CBUFFER_START(UnityPerMaterial)
    float4 _BaseMap_ST;
    float4 _HeightMap_ST;
    half4 _BaseColor;
    half _ParallaxScale;
CBUFFER_END
```

- Unity 将每个材质的常量数据放在 `UnityPerMaterial` 缓冲区中。
- `_BaseMap_ST` 和 `_HeightMap_ST` 是 Unity 为纹理自动提供的缩放/平移值：`x, y` 是缩放，`z, w` 是平移。
- `_BaseColor` 和 `_ParallaxScale` 与材质 Inspector 的属性名对应。
- `float` 和 `half` 都是浮点数类型；`half` 通常用于精度要求较低的材质值或向量。

### 3.4 顶点输入和阶段间数据

```hlsl
struct Attributes
{
    float4 positionOS : POSITION;
    float3 normalOS : NORMAL;
    float4 tangentOS : TANGENT;
    float2 uv : TEXCOORD0;
};
```

这个结构说明 Unity 交给顶点程序的数据：

- `positionOS` 是物体空间（Object Space）的顶点位置；`POSITION` 是 Unity/HLSL 约定的网格输入语义。
- `normalOS` 是物体空间法线；`NORMAL` 表示网格法线。
- `tangentOS` 是物体空间切线；`TANGENT` 表示网格切线，额外的 `w` 分量保存切线坐标系的方向符号。
- `uv` 是 UV0；`TEXCOORD0` 表示网格的第一组纹理坐标。

```hlsl
struct Varyings
{
    float4 positionCS : SV_POSITION;
    float2 uv : TEXCOORD0;
    half3 viewDirTS : TEXCOORD1;
};
```

这个结构说明顶点程序传给片元程序的数据：

- `positionCS` 是裁剪空间（Clip Space）的位置。`SV_POSITION` 是 GPU 使用的系统语义，负责把顶点放到屏幕上。
- `uv` 沿网格表面插值后传到片元阶段。
- `viewDirTS` 是切线空间观察方向；这里用 `TEXCOORD1` 作为阶段间插值通道。名字叫 `TEXCOORD`，也可以传普通向量，不只传贴图坐标。

### 3.5 顶点函数 `Vert`

```hlsl
Varyings Vert(Attributes input)
{
    Varyings output;
```

顶点入口接收一份 `Attributes`，并创建一份要返回给片元阶段的 `Varyings`。

```hlsl
VertexPositionInputs positionInputs = GetVertexPositionInputs(input.positionOS.xyz);
VertexNormalInputs normalInputs = GetVertexNormalInputs(input.normalOS, input.tangentOS);
```

- `GetVertexPositionInputs` 将物体空间顶点位置变换为世界空间、观察空间和裁剪空间等坐标。
- `GetVertexNormalInputs` 根据法线和切线，计算世界空间的法线、切线和副切线。
- `.xyz` 只取位置向量，不传入 `float4` 的齐次分量。

```hlsl
output.positionCS = positionInputs.positionCS;
output.uv = input.uv;
```

将裁剪空间位置交给 GPU，并把原 UV 继续传给片元阶段。

```hlsl
half3 viewDirWS = GetWorldSpaceViewDir(positionInputs.positionWS);
```

URP 函数根据当前顶点世界位置计算指向相机的世界空间方向。`WS` 是 World Space（世界空间）的缩写。

```hlsl
half3x3 tangentToWorld = half3x3(
    normalInputs.tangentWS,
    normalInputs.bitangentWS,
    normalInputs.normalWS);
```

用世界空间切线、副切线和法线组成切线到世界空间的变换矩阵。三根轴决定如何在这两种坐标系间转换。

```hlsl
output.viewDirTS = TransformWorldToTangentDir(viewDirWS, tangentToWorld, true);
return output;
```

URP 函数把观察方向从世界空间转成切线空间；最后的 `true` 请求归一化结果。返回的 `output` 经过 GPU 插值后会送到片元入口。

### 3.6 基础视差函数 `ApplyParallax`

```hlsl
half2 ApplyParallax(half2 uv, half3 viewDirTS)
{
    half2 heightUV = TRANSFORM_TEX(uv, _HeightMap);
```

函数输入当前 UV 和切线空间观察方向，输出偏移后的 UV。`TRANSFORM_TEX` 应用材质面板中高度图的 Tiling（缩放）和 Offset（平移）。

```hlsl
half height = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, heightUV).r;
```

用高度图采样器读取当前高度图像素，并取红色通道作为高度值。

```hlsl
half viewZ = max(viewDirTS.z, 0.08h);
half2 viewOffset = viewDirTS.xy / viewZ;
```

限制法线方向分量的最小值，避免掠射角导致除零或过大偏移；然后计算观察方向在表面平面上的斜率。`xy` 是切线、副切线方向，`z` 是法线方向。

```hlsl
half centeredHeight = height - 0.5h;
half2 parallaxOffset = viewOffset * centeredHeight * _ParallaxScale;
return uv - parallaxOffset;
```

将高度以中灰为零点，乘观察方向斜率与材质强度得到 UV 偏移。函数返回平移后的 UV；用减号还是加号取决于所采用的观察方向约定，这份代码统一使用减号。

### 3.7 片元函数 `Frag`

```hlsl
half4 Frag(Varyings input) : SV_Target
{
    half2 parallaxUV = ApplyParallax(input.uv, normalize(input.viewDirTS));
```

片元入口接收插值后的 `Varyings`。`normalize` 再次确保插值后的观察方向长度为 1。`SV_Target` 表示返回值写到当前颜色目标。

```hlsl
half2 baseUV = TRANSFORM_TEX(parallaxUV, _BaseMap);
half4 color = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, baseUV) * _BaseColor;
```

将偏移后的 UV 变换到颜色图自身的 Tiling/Offset，然后采样颜色图并乘材质颜色。

```hlsl
return half4(color.rgb, 1.0h);
```

返回纹理的 RGB 颜色，并把透明度设为 1。因为此材质声明为不透明，所以不会使用颜色图 Alpha 做透明混合。

### 3.8 额外的深度 Pass

```hlsl
Pass
{
    Name "DepthOnly"
    Tags { "LightMode"="DepthOnly" }
    ZWrite On
    ColorMask 0
    Cull Back
```

URP 需要深度预通道时，可以调用这个 Pass。`ColorMask 0` 禁止写入颜色，只保留深度写入；`Cull Back` 仍剔除背面。

```hlsl
#pragma vertex DepthVert
#pragma fragment DepthFrag
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
```

为深度 Pass 指定自己的顶点、片元入口，并引入 URP 的坐标函数。

```hlsl
struct Attributes { float4 positionOS : POSITION; };
struct Varyings { float4 positionCS : SV_POSITION; };
```

这个 Pass 只需顶点位置，不需要 UV、法线或切线。输出仅需裁剪空间位置。

```hlsl
Varyings DepthVert(Attributes input)
{
    Varyings output;
    output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
    return output;
}
```

顶点函数把物体空间位置变到裁剪空间。`HClip` 表示齐次裁剪空间坐标。

```hlsl
half4 DepthFrag(Varyings input) : SV_Target { return 0; }
```

片元返回值不写入颜色，因为 `ColorMask 0` 已关闭颜色写入；GPU 仍会按 Pass 状态写深度。这个深度来自原始网格，不包含视差偏移。

### 3.9 额外的法线 Pass

```hlsl
Pass
{
    Name "DepthNormals"
    Tags { "LightMode"="DepthNormals" }
    ZWrite On
    Cull Back
```

URP 的某些效果（例如屏幕空间环境光遮蔽）需要深度和法线数据，因此 Shader 可以提供 `DepthNormals` Pass。

```hlsl
#pragma vertex DepthNormalsVert
#pragma fragment DepthNormalsFrag
#pragma multi_compile_fragment _ _GBUFFER_NORMALS_OCT
```

指定本 Pass 的入口。`multi_compile_fragment` 让 Unity 为普通法线输出和八面体压缩法线输出分别编译变体；URP 可按当前目标缓冲格式选择。

输入结构包含 `POSITION`、`NORMAL`、`TANGENT`，输出结构包含裁剪空间位置和世界空间法线。顶点函数用 `TransformObjectToHClip` 变换位置，用 `GetVertexNormalInputs` 得到世界空间法线，再通过 `NormalizeNormalPerVertex` 归一化并传下去。

片元函数在 `_GBUFFER_NORMALS_OCT` 开启时，先将法线八面体编码，再映射到 `[0,1]` 并打包到颜色通道；否则直接输出归一化的世界空间法线。两条路径都把 Alpha 设为 0。这里传出的仍是网格法线，不是 POM 推算出的纹理表面法线。

这两个 Pass 让当前学习场景中的深度相关效果能识别网格，但它们都不执行视差搜索。完整生产 Shader 还可能需要阴影投射和烘焙等 Pass；是否需要取决于项目功能。

## 4. 在 Unity 中使用基础版本

1. 打开 `Assets/Scenes/SampleScene.unity`。
2. 将 `M_04_CodeParallax` 赋给具有 UV0 和有效切线的网格。
3. 确认 Base Map 使用 `ParallaxTiles_Albedo`，Height Map 使用 `ParallaxTiles_Height`。高度图应作为线性数据导入；项目已经关闭其 sRGB 转换。
4. 从较小的 Parallax Scale 开始，慢慢旋转视角。把强度设为 0 就会回到普通纹理采样。
5. 试着换成没有切线或 UV 展开的模型，观察为什么视差结果会错误。

## 5. 进阶版：在项目中实现 POM

新文件 `Assets/RenderingLearning/Shaders/HLSL/05_CodeParallaxOcclusionUnlit.shader` 保留 URP Unlit 的颜色输出流程，将单次高度偏移替换为高度射线步进。`M_05_CodeParallaxOcclusion.mat` 复用现有颜色图和高度图，便于与 `M_04_CodeParallax` 并排比较。

关键实现会完成以下工作：

1. 把观察方向传入片元阶段，并限制 `Vz`，避免斜视时无限放大偏移。
2. 依据观察角度，在最少步数和最多步数之间选取采样次数；越斜视就越多采样。
3. 沿射线逐层读取高度图，找到深度差从负变正的区间。
4. 在区间两端之间插值估算交点，再用交点 UV 读取颜色图。

POM 比基础版本需要更多纹理采样，低端设备或大面积材质上应减少最大步数，或者对远处材质关闭效果。它仍然不是真正的几何位移；示例的 `DepthOnly`、`DepthNormals` Pass 仍描述原始网格表面。

## 6. 方法一：让 POM 响应场景光照

`05` 的片元程序直接返回贴图颜色，所以它是 Unlit：场景中的灯光不会参与颜色计算。Lit Shader 则把表面信息交给 URP 的光照函数，由 URP 计算主光、额外灯、环境光和材质反射。

### 6.1 Lit 需要哪些输入？

URP PBR（基于物理的渲染）至少需要颜色、表面法线、观察方向、世界位置和光照/阴影数据。`06_CodeParallaxOcclusionLit.shader` 会在 POM 找到的 UV 处读取颜色和高度，再把构造出的 `SurfaceData` 与 `InputData` 交给 `UniversalFragmentPBR`。

仅把网格法线交给 Lit Shader，整块平面仍会像平板一样受光。`06` 因而从高度图附近的采样值估算切线空间法线。令 `hL`、`hR`、`hD`、`hU` 分别表示交点 UV 左、右、下、上邻近位置的高度，令 `k` 表示法线强度，则示意计算为：

```text
N_TS = normalize((-(hR - hL) × k, -(hU - hD) × k, 1))
N_WS = 切线空间到世界空间的变换(N_TS)
```

`N_TS` 是切线空间法线，`N_WS` 是世界空间法线；`normalize` 将向量长度变为 1。高度图的灰度变化越陡，法线偏转越明显。这里是用高度图估算法线，不等于真实几何曲面，也可能需要调小 `Normal Strength` 来减少锯齿或闪烁。

`06` 在交点 UV 两侧各读取一次高度，估算两个方向的梯度，再把法线变换到世界空间。核心代码可以缩写为：

```hlsl
half hL = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, heightUV - half2(texel.x, 0)).r;
half hR = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, heightUV + half2(texel.x, 0)).r;
half hD = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, heightUV - half2(0, texel.y)).r;
half hU = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, heightUV + half2(0, texel.y)).r;
half3 normalTS = SafeNormalize(half3((hL - hR) * _NormalStrength,
                                     (hD - hU) * _NormalStrength, 1.0h));
half3 normalWS = TransformTangentToWorld(normalTS, tangentToWorld, true);
```

随后，Shader 将颜色、法线和位置等数据交给 URP：

```hlsl
lightingInput.positionWS = input.positionWS;
lightingInput.normalWS = normalWS;
lightingInput.shadowCoord = TransformWorldToShadowCoord(input.positionWS);
surface.albedo = baseSample.rgb;
surface.normalTS = half3(0, 0, 1); // 法线已经放进 lightingInput.normalWS
return UniversalFragmentPBR(lightingInput, surface);
```

`InputData` 描述当前片元的位置、法线、观察方向和阴影坐标；`SurfaceData` 描述颜色、金属度、光滑度等材质属性。URP 的 PBR 函数计算最终颜色。

```text
POM UV → 颜色/高度采样 → 高度梯度法线 → URP PBR → 像素颜色
```

### 6.2 URP 的普通场景阴影

`06` 使用 URP PBR 光照函数，并加入 `ShadowCaster` Pass。这样可以让平面接收 URP 阴影，也可以把**网格本身**登记到灯光的阴影贴图中。URP 的自定义光照 Shader 需要读取光照/阴影数据；`ShadowCaster` Pass 则负责参与阴影贴图绘制。[Unity 自定义 URP Shader 的光照接口](https://docs.unity3d.com/cn/6000.0/Manual/urp/use-built-in-shader-methods-lighting.html) 与 [URP Lit Shader 的阴影 Pass](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.universal/Shaders/Lit.shader) 展示了这两类接口。

`ShadowCaster` Pass 的关键是 `LightMode` 标签和 URP 提供的阴影绘制程序：

```hlsl
Pass
{
    Name "ShadowCaster"
    Tags { "LightMode"="ShadowCaster" }
    ZWrite On
    ColorMask 0
    HLSLPROGRAM
    #pragma vertex ShadowPassVertex
    #pragma fragment ShadowPassFragment
    #pragma multi_compile_vertex _ _CASTING_PUNCTUAL_LIGHT_SHADOW
    #include "Packages/com.unity.render-pipelines.universal/Shaders/ShadowCasterPass.hlsl"
    ENDHLSL
}
```

URP 会从灯光视角绘制这个物体。这个 Pass 使用原始网格位置，所以投出的仍是网格形状的阴影。

由于 ShadowCaster 处理的仍是原始网格，平面投射的轮廓仍是平的。高度图上的砖块、石块等凸起不会因此在附近的其他物体上投出细节阴影；下一节的自阴影只处理 POM 表面内部的遮挡。

### 6.3 在 Unity 中比较

把 `M_05_CodeParallaxOcclusion`、`M_06_CodeParallaxOcclusionLit` 放在相同网格上。转动或关闭 Directional Light：`05` 的颜色基本不变，`06` 会随场景光照变化。将物体放到其他物体旁并启用灯光阴影，可以观察 `06` 的普通 ShadowCaster 效果；阴影轮廓仍跟随网格。

## 7. 方法二：沿灯光方向计算高度图自阴影

普通场景阴影依据灯光阴影贴图中的网格深度。**高度图自阴影**则在 POM 已求得视线交点后，再从该交点沿灯光方向穿过高度图。如果射线先遇到更高的高度表面，该交点就被高度图内部的凸起挡住。

### 7.1 第二条射线

定义：

- `u_h`、`t_h`：视线 POM 找到的交点 UV 和归一化深度。
- `L = (Lx, Ly, Lz)`：从交点指向灯光的单位方向，已变换到切线空间。
- `s`：POM 的高度范围强度。
- `t ∈ [0, t_h]`：自阴影射线从交点往表面顶部移动时的当前深度。
- `u_L(t) = u_h + (Lx, Ly) / max(Lz, 0.08) × s × (t_h - t)`：灯光射线在深度 `t` 处对应的高度图 UV。
- `D(u) = 1 - h(u)`：高度图在 UV `u` 的表面深度。
- `F_L(t) = t - D(u_L(t))`：灯光射线位置与高度表面的深度差。

当射线离开交点后，若在某个采样位置有 `F_L(t) > bias`，代表它已进入高度表面内部，也就是在到达灯光前被凸起挡住。`bias` 是很小的容差，用来减少交点附近的误判。若所有采样点均未被挡住，这个表面点对该灯光可见。

```text
hitUV, hitDepth = 沿视线执行 POM
对主光和每个逐像素额外灯：
    把灯光方向变换到切线空间
    从 hitUV、hitDepth 沿灯光方向分步采样高度图
    若射线与高度表面相交：selfVisibility = 0
    否则：selfVisibility = 1
    directLight *= selfVisibility
输出间接光 + 经普通阴影和自阴影调制的直接光
```

`07` 的射线步进核心如下：

```hlsl
half travel = (i + 1) * depthStep;
half rayDepth = hitDepth - travel;
half2 rayUV = hitUV + lightSlope * _ParallaxScale * travel;
half surfaceDepth = 1.0h - SAMPLE_TEXTURE2D(
    _HeightMap, sampler_HeightMap, TRANSFORM_TEX(rayUV, _HeightMap)).r;
if (rayDepth > surfaceDepth + _SelfShadowBias)
    return 1.0h - _SelfShadowStrength;
```

`travel` 控制射线离开交点的距离；`rayDepth` 沿表面方向逐渐减小；`rayUV` 随灯光方向移动。若射线已经进入高度表面内部，就按 `Self Shadow Strength` 降低直接光。

`07_CodeParallaxOcclusionSelfShadow.shader` 采用这条第二射线，并将自阴影乘到主光和逐像素额外灯的直接光贡献上；环境光、烘焙间接光不会被这条射线错误地整体压暗。为了控制自阴影对直接光的影响，`07` 在 URP PBR 的直接光计算中插入可见度系数：

```hlsl
half selfVisibility = ComputePOMSelfShadow(hitUV, hitDepth, lightDirTS);
lightingData.mainLightColor = LightingPhysicallyBased(
    brdfData, brdfDataClearCoat, mainLight,
    inputData.normalWS, inputData.viewDirectionWS,
    surface.clearCoatMask, specularHighlightsOff) * selfVisibility;
```

真实 URP 阴影贴图仍用于场景物体之间的普通遮挡。顶点阶段汇总的额外灯不进行这项逐像素自阴影查询。POM 自阴影的射线步数越多，阴影边缘越稳定，但每个参与查询的灯光也会增加高度图采样。

### 7.2 在 Unity 中比较

在同一平面上依次使用 `M_06_CodeParallaxOcclusionLit` 与 `M_07_CodeParallaxOcclusionSelfShadow`，移动主光方向并从斜角观察。`06` 有场景光照和普通阴影；`07` 额外显示高度图凸起遮住凹处的细节自阴影。先用较低的 `Self Shadow Steps` 观察效果，再逐步加高观察阴影质量。

自阴影只影响本材质高度图内部的直接光可见度。它不会将纹理细节变成真实几何，也不会在邻近物体表面生成对应的砖块/石块投影。POM 和近似软自阴影的技术基础可参考 Tatarchuk 的研究。[SIGGRAPH 2006 论文记录](https://doi.org/10.1145/1185657.1185830)

## 8. 常见问题

| 现象 | 优先检查 |
| --- | --- |
| 完全看不出视差 | 强度是否为 0；高度贴图是否绑定；观察角度是否太正；纹理高度是否有对比度 |
| 偏移方向相反 | 高度图黑白约定、视线方向定义和 UV 加减号是否一致 |
| 斜视时跳动或拉伸 | 降低强度；检查切线；检查是否需要 UV 边界裁剪；增加步数会改善 POM 精度但增加成本 |
| 只有部分模型正确 | 网格是否有 UV0 与切线；切线是否与 UV 展开相匹配 |
| 环境光遮蔽或深度边缘不匹配 | 深度/法线 Pass 使用的是原始网格几何，并未包含 POM 的虚拟表面 |

## 参考资料

- Unity URP 13.1.9 项目资源：`Packages/manifest.json`、`Assets/RenderingLearning/Shaders/HLSL/04_CodeParallaxUnlit.shader` 与 `M_04_CodeParallax.mat`。
- Unity URP 13.1.9 项目 Lit 示例：`Assets/RenderingLearning/Shaders/HLSL/02_CodeLitTexture.shader`；新增视差 Lit 与自阴影示例位于 `06_CodeParallaxOcclusionLit.shader` 和 `07_CodeParallaxOcclusionSelfShadow.shader`。
- Unity Graphics 文档：[URP ShaderLab Pass Tags](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@13.1/manual/urp-shaderlab-pass-tags.html)。
- Unity Graphics 源码文档：[Core RP built-in shader methods](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.core/Documentation~/built-in-shader-methods.md)，包括 `TransformWorldToTangentDir` 的输入、输出与用途。
- Natalya Tatarchuk, [Practical Parallax Occlusion Mapping with Approximate Soft Shadows for Detailed Surface Rendering](https://advances.realtimerendering.com/s2006/Tatarchuk-POM.pdf), 2006。POM 的逐层高度射线追踪及相关改进的技术资料。
