# Compute Shader学习笔记（三）之 草地渲染

项目地址：https://github.com/Remyuu/Unity-Compute-Shader-Learn

![image-20240604171102956](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604171102956.png)

## L5 草地渲染

当前做的效果非常丑陋，还有很多细节没有完善，仅仅是“实现”了。

知识点小结：

- 草地渲染方案
- `UNITY_PROCEDURAL_INSTANCING_ENABLED`
- `bounds.extents`
- 射线检测
- 罗德里格旋转
- 四元数旋转

### 前言1

> 前言参考文章：
>
> https://danielilett.com/2022-12-05-tut6-2-six-grass-techniques/
>
> https://developer.nvidia.com/gpugems/gpugems/part-i-natural-effects/chapter-7-rendering-countless-blades-waving-grass
>
> https://pixelantgames.com/blog/rendering-millions-of-grass-blades-using-ue-and-niagara/

![image-20240531165537056](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531165537056.png)

草地渲染有很多方法。

最简单的是直接一张草地的纹理贴上去。

![image-20240531174757953](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531174757953.png)

除此之外，将一个个**Mesh草**拖到场景中也很常见。这种方法操作空间大，每一颗草都在掌控中。虽然可以用Batching等方法优化，减少CPU到GPU的传输时间，但是这会损耗您键盘上的Ctrl、C、V和D键的寿命。不过可以在Transform组件里面用 `L(a, b)` 让选中的物体平均分布在 `a` 和 `b` 之间。想随机，可以用 `R(a, b)` 。更多相关的操作可以看[官方文档](https://docs.unity3d.com/Manual/EditingValueProperties.html)。

![image-20240531171142602](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531171142602.png)

还可以结合**几何着色器和曲面细分着色器**，这个方法看起来不错的，但是一个着色器只能对应一种几何（草），如果想要在这个网格生成花或者岩石，就需要在几何着色器中修改代码。这个问题其实不是最关键的，更要命的问题是很多移动设备还有Metal根本就不支持几何着色器，就算支持也只是软件模拟的，性能差劲。并且每一帧都会重新计算一次草地Mesh，浪费性能。

![image-20240531170045756](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531170045756.png)

**广告牌**技术渲染草也是一种广泛流传经久不衰的方法。当我们不需要高保真的画面时，这个方法非常奏效。这个方法是简单的渲一个Quad+贴图（Alpha裁切）。用DrawProcedural就可以了。但是这个方法只可远观不可近看，否则就会大露馅。

![image-20240531172522186](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531172522186.png)

用Unity的**地形系统**也可以画出非常nice的草。并且Unity使用了instancing技术确保了性能。其中最好用的地方莫过于他的笔刷工具，但是如果你的工作流没有地形系统的身影，那么你还可以用第三方插件做到。

![image-20240531172828086](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531172828086.png)

在搜索资料的时候我还发现了一种叫**Impostors「冒名顶替」技术**。结合了广告牌的顶点节省优势和从多个角度真实重现对象的能力，还挺有意思。这个技术通过预先从多个角度“拍下”一个真实草的Mesh照片，通过Texture存起来。运行的时候根据当前相机的观看方向选择合适的纹理进行渲染。相当于广告牌技术的升级版。我认为Impostors技术非常适合用于那些大型但玩家可能需要从多个角度查看的对象，如树木或复杂建筑。然而，当相机非常接近或者在两个角度之间变换时，这种方法可能会出现问题。比较合理的方案是：在距离非常近用基于Mesh的方法，中等距离用Impostors，远距离用广告牌。

![image-20240531173615877](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531173615877.png)

本文要实现的方法是基于GPU Instancing的，应该称之为「**per-blade mesh grass**」。在《對馬島之魂》、《原神》和《薩爾達傳說：曠野之息》等游戏上都是使用这种方案。每个草都有自己的实体，光影效果也相当真实。



![image-20240531175324823](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531175324823.png)

渲染流程：

![image-20240531175425570](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531175425570.png)

### 前言2

Unity的Instancing技术比较复杂，我也只是管中窥豹，出现错误请指正。目前的代码都是仿照文档写的。GPU instancing目前支持的平台：

- Windows: DX11 and DX12 with SM 4.0 and above / OpenGL 4.1 and above
- OS X and Linux: OpenGL 4.1 and above
- Mobile: OpenGL ES 3.0 and above / Metal
- PlayStation 4
- Xbox One

另外`Graphics.DrawMeshInstancedIndirect`目前已经淘汰了，应该使用 `Graphics.RenderMeshIndirect`  ，这个函数会自动计算Bounding Box，这个就是后话了。详细请看官方文档：[RenderMeshIndirect](https://docs.unity.cn/ScriptReference/Graphics.RenderMeshIndirect.html) 。这篇文章也很有帮助https://zhuanlan.zhihu.com/p/403885438。

GPU Instancing原理是将多个具有相同Mesh的对象发一次Draw Call。CPU首先收集好所有信息，然后放到数组里一次性发给GPU。局限就是这些对象的Material和Mesh都要相同。这就是一次能绘制这么多草而保持高性能的原理。要实现GPU Instancing绘制上百万的Mesh，就需要遵循一些规定：

- 所有的网格需使用相同的Material
- 勾选GPU Instancing
- Shader需支持实例化
- 不支持Skin Mesh Renderer

由于不支持Skin Mesh Renderer，在[上一篇文章中](https://zhuanlan.zhihu.com/p/700370323)，我们绕过了SMR，直接取了不同关键帧的Mesh出来传给GPU，这也是上一篇文章最后提出那个问题的原因。

Unity中的Instancing分为两种主要类型：GPU Instancing和Procedural Instancing（涉及到Compute Shaders和Indirect Drawing技术），还有一种是立体渲染路径（`UNITY_STEREO_INSTANCING_ENABLED`），这里就不深入了。在Shader中，前者用`#pragma multi_compile_instancing` 后者用`#pragma instancing_options procedural:setup` 。具体的请看官方文档[Creating shaders that support GPU instancing](https://docs.unity.cn/cn/2023.2/Manual/gpu-instancing-shader.html)  。

然后目前SRP管线不支持自定义的GPU Instancing Shader，只有BIRP可以。

然后就是`UNITY_PROCEDURAL_INSTANCING_ENABLED` 。这个宏用于表示是否启用了Procedural Instancing。在使用Compute Shader或Indirect Drawing API时，实例的属性（如位置、颜色等）可以在GPU上实时计算并直接用于渲染，无需CPU的介入。在[源代码中](https://github.com/TwoTailsGames/Unity-Built-in-Shaders/blob/master/CGIncludes/UnityInstancing.cginc)，关于这个宏的核心代码是：

```glsl
#ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    #ifndef UNITY_INSTANCING_PROCEDURAL_FUNC
        #error "UNITY_INSTANCING_PROCEDURAL_FUNC must be defined."
    #else
        void UNITY_INSTANCING_PROCEDURAL_FUNC(); // 前向声明程序化函数
        #define DEFAULT_UNITY_SETUP_INSTANCE_ID(input)      { UnitySetupInstanceID(UNITY_GET_INSTANCE_ID(input)); UNITY_INSTANCING_PROCEDURAL_FUNC();}
    #endif
#else
    #define DEFAULT_UNITY_SETUP_INSTANCE_ID(input)          { UnitySetupInstanceID(UNITY_GET_INSTANCE_ID(input));}
#endif
```

要求Shader定义一个`UNITY_INSTANCING_PROCEDURAL_FUNC`函数，其实就是 `setup()` 函数。没有这个`setup()`函数，就会报错。

一般来说，`setup()`函数要做的就是从Buffer中取出对应（`unity_InstanceID`） 的数据，然后计算当前实例的位置、变换矩阵、颜色、金属度或者是自定义数据等属性。

GPU Instancing只是Unity众多优化手段的一种，仍然需要继续学习。

### 1. 摇曳的3-Quad草

这一章所运用关于CS的知识点在上一篇文章都已全部涉及，只不过换一个背景罢了。简单画一个示意图。

![image-20240531152635013](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531152635013.png)

实现是使用GPU Instancing，也就是一次性渲染一大片Mesh。核心的代码就一句：

```csharp
Graphics.DrawMeshInstancedIndirect(mesh, 0, material, bounds, argsBuffer);
```

Mesh采用三个Quad共六个三角形组成。

![image-20240531135902033](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531135902033.png)

然后上一张贴图+Alpha Test。

![image-20240531141914478](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531141914478.png)

草的数据结构：

- 位置
- 倾斜角度
- 随机噪声值（用于计算随机的倾斜角度）

```csharp
public Vector3 position; // 世界坐标，需要计算
public float lean;
public float noise;

public GrassClump( Vector3 pos){
    position.x = pos.x;
    position.y = pos.y;
    position.z = pos.z;
    lean = 0;
    noise = Random.Range(0.5f, 1);
    if (Random.value < 0.5f) noise = -noise;
}
```

将需要渲染的草的Buffer（世界坐标需要计算）传给GPU。首先确定草在哪里生成、生成多少。获取当前物体的Mesh（暂时假设是一个Plane Mesh）的AABB。

```csharp
Bounds bounds = mf.sharedMesh.bounds;
Vector3 clumps = bounds.extents;
```

![image-20240531144201324](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531144201324.png)

确定草的范围，然后在xOz平面上随机生成草。

![image-20240531144136093](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531144136093.png)

需要注意，当前还是在物体空间，因此需要将Object Space转换到World Space。

```csharp
pos = transform.TransformPoint(pos);
```

再结合密度`density`参数和物体缩放系数，计算出一共要渲染多少个草。

```csharp
Vector3 vec = transform.localScale / 0.1f * density;
clumps.x *= vec.x;
clumps.z *= vec.z;
int total = (int)clumps.x * (int)clumps.z;
```

由于Compute Shader的逻辑是每个线程计算一棵草，极有可能需要渲染的草的数量不是线程的倍数。因此将需要渲染的草的数量向上取整到线程的倍数。也就是说，当密度因子=1的时候，渲染的草的数量等于一个线程组中线程的数量。

```csharp
groupSize = Mathf.CeilToInt((float)total / (float)threadGroupSize);
int count = groupSize * (int)threadGroupSize;
```

让Compute Shader计算每个草的倾斜角度。

```glsl
GrassClump clump = clumpsBuffer[id.x];
clump.lean = sin(time) * maxLean * clump.noise;
clumpsBuffer[id.x] = clump;
```

将草的位置、旋转角度传给GPU Buffer还没完，还得拜托Material决定渲染实例的最终外观，才能最终执行`Graphics.DrawMeshInstancedIndirect`。

渲染流程中，在实例化阶段之前（也就是`procedural:setup`函数内），使用`unity_InstanceID`确定现在渲的是哪个草。获取当前草的世界空间，草的倾倒值。

```glsl
GrassClump clump = clumpsBuffer[unity_InstanceID];
_Position = clump.position;
_Matrix = create_matrix(clump.position, clump.lean);
```

具体的旋转+位移矩阵：

```glsl
float4x4 create_matrix(float3 pos, float theta){
    float c = cos(theta); // 计算旋转角度的余弦值
    float s = sin(theta); // 计算旋转角度的正弦值
    // 返回一个4x4变换矩阵
    return float4x4(
        c, -s, 0, pos.x, // 第一行：X轴旋转和位移
        s,  c, 0, pos.y, // 第二行：Y轴旋转（对于2D足够，但草丛可能不使用）
        0,  0, 1, pos.z, // 第三行：Z轴不变
        0,  0, 0, 1     // 第四行：均匀坐标（保持不变）
    );
}
```

这个公式怎么推的呢？将(0,0,1)带入罗德里格斯公式得到一个$$3*3$$的旋转矩阵，然后扩展到重心坐标。带入 $$x=0,y=0,z=1$$​ 就是代码的公式了。
$$
t x^2 + c & t x y - s z & t x z + s y \\
t x y + s z & t y^2 + c & t y z - s x \\
t x z - s y & t y z + s x & t z^2 + c
$$
用这个矩阵乘上Object Space的顶点，得到倾倒+位移的顶点坐标。

```glsl
v.vertex.xyz *= _Scale;
float4 rotatedVertex = mul(_Matrix, v.vertex);
v.vertex = rotatedVertex;
```

这时候问题来了。目前草并不是一个平面，而是三组Quad组成的立体图形。

![image-20240531162917792](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531162917792.png)

如果简单的将所有顶点按照z轴旋转，就会出现草根大偏移的问题。

![image-20240531162513432](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240531162513432.png)

因此借助 `v.texcoord.y` ，将旋转前后的顶点位置lerp起来。这样，纹理坐标的Y值越高（即顶点在模型上的位置越靠近顶部），顶点受到的旋转影响就越大。由于草根的Y值为0，lerp之后草根就不会乱晃了。

```glsl
v.vertex.xyz *= _Scale;
float4 rotatedVertex = mul(_Matrix, v.vertex);
// v.vertex = rotatedVertex;
v.vertex.xyz += _Position;
v.vertex = lerp(v.vertex, rotatedVertex, v.texcoord.y);
```

效果很差，草太假了。这种Quad草只有在远处用用。

- 摆动僵硬
- 叶片僵硬
- 光影效果很差

![image-20240603120326912](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240603120326912.png)

当前版本代码：

- 脚本：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_3QuadGrass/Assets/Scripts/GrassClumps.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_3QuadGrass/Assets/Shaders/GrassClumps.shader
- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_3QuadGrass/Assets/Shaders/GrassClumps.compute

### 2. 程式化草叶

上一节用几个Quad和带Alpha贴图的草，用sin wave做扰动，效果非常一般。现在用程式化的草和噪声贴图改善。

在 C# 中定义草的顶点、法线和uv作为Mesh传到GPU上。

```csharp
Vector3[] vertices =
{
    new Vector3(-halfWidth, 0, 0),
    new Vector3( halfWidth, 0, 0),
    new Vector3(-halfWidth, rowHeight, 0),
    new Vector3( halfWidth, rowHeight, 0),
    new Vector3(-halfWidth*0.9f, rowHeight*2, 0),
    new Vector3( halfWidth*0.9f, rowHeight*2, 0),
    new Vector3(-halfWidth*0.8f, rowHeight*3, 0),
    new Vector3( halfWidth*0.8f, rowHeight*3, 0),
    new Vector3( 0, rowHeight*4, 0)
};
Vector3 normal = new Vector3(0, 0, -1);
Vector3[] normals =
{
	normal, normal, normal, normal, normal, normal, normal, normal, normal
};
Vector2[] uvs =
{
    new Vector2(0,0),
    new Vector2(1,0),
    new Vector2(0,0.25f),
    new Vector2(1,0.25f),
    new Vector2(0,0.5f),
    new Vector2(1,0.5f),
    new Vector2(0,0.75f),
    new Vector2(1,0.75f),
    new Vector2(0.5f,1)
};
```

Unity的Mesh还有一个顶点顺序需要设定，默认是**逆时针**。如果顺时针写并且开启背面剔除，那就啥也看不见了。

![image-20240603124313298](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240603124313298.png)

```csharp
int[] indices =
{
    0,1,2,1,3,2,//row 1
    2,3,4,3,5,4,//row 2
    4,5,6,5,7,6,//row 3
    6,7,8//row 4
};
mesh.SetIndices(indices, MeshTopology.Triangles, 0);
```

在代码那边设置好风的方向、大小还有噪声比重，打包进一个float4里面，传给Compute Shader计算一片草叶的摆动方向。

```csharp
Vector4 wind = new Vector4(Mathf.Cos(theta), Mathf.Sin(theta), windSpeed, windScale);
```

一个草叶的数据结构

```csharp
struct GrassBlade
{
    public Vector3 position;
    public float bend; // 随机草叶倾倒
    public float noise;// CS计算噪声值
    public float fade; // 随机草叶明暗
    public float face; // 叶片朝向

    public GrassBlade( Vector3 pos)
    {
        position.x = pos.x;
        position.y = pos.y;
        position.z = pos.z;
        bend = 0;
        noise = Random.Range(0.5f, 1) * 2 - 1;
        fade = Random.Range(0.5f, 1);
        face = Random.Range(0, Mathf.PI);
    }
}
```

![image-20240603191709495](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240603191709495.png)

当前的草叶都是一个方向的。Setup函数里，先修改叶片朝向。

```glsl
// 创建绕Y轴的旋转矩阵（面向）
float4x4 rotationMatrixY = AngleAxis4x4(blade.position, blade.face, float3(0,1,0));
```

![image-20240603191900360](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240603191900360.png)

将草叶倾倒的逻辑（由于`AngleAxis4x4`是包含了位移，下图只是单独演示了叶片倾倒而没有随机朝向，如果要得到下图的效果代码中记得加入位移）：

```glsl
// 创建绕X轴的旋转矩阵（倾倒）
float4x4 rotationMatrixX = AngleAxis4x4(float3(0,0,0), blade.bend, float3(1,0,0));
```

![image-20240603192105194](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240603192105194.png)

然后合成两个旋转矩阵。

```glsl
_Matrix = mul(rotationMatrixY, rotationMatrixX);
```

![image-20240603192337322](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240603192337322.png)

现在的光照是非常奇怪的。因为法线没有修改。

```glsl
// 计算逆转置矩阵用于法线变换
float3x3 normalMatrix = (float3x3)transpose(((float3x3)_Matrix));
// 变换法线
v.normal = mul(normalMatrix, v.normal);
```

这里逆矩阵的代码：

```glsl
float3x3 transpose(float3x3 m)
{
    return float3x3(
        float3(m[0][0], m[1][0], m[2][0]), // Column 1
        float3(m[0][1], m[1][1], m[2][1]), // Column 2
        float3(m[0][2], m[1][2], m[2][2])  // Column 3
    );
}
```

为了代码可读性，再补上齐次坐标变换矩阵，这里升级为那个著名的旋转公式：

```glsl
float4x4 AngleAxis4x4(float3 pos, float angle, float3 axis){
    float c, s;
    sincos(angle*2*3.14, s, c);

    float t = 1 - c;
    float x = axis.x;
    float y = axis.y;
    float z = axis.z;

    return float4x4(
        t * x * x + c    , t * x * y - s * z, t * x * z + s * y, pos.x,
        t * x * y + s * z, t * y * y + c    , t * y * z - s * x, pos.y,
        t * x * z - s * y, t * y * z + s * x, t * z * z + c    , pos.z,
        0,0,0,1
        );
}
```

![image-20240603191104957](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240603191104957.png)

![image-20240603190146147](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240603190146147.png)

![image-20240603200021604](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240603200021604.png)

想要在不平坦的地面生成怎么办？

![image-20240604113536389](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240604113536389.png)

只需要修改生成草地初始位置高度的逻辑，用MeshCollider加射线检测，

```csharp
bladesArray = new GrassBlade[count];
gameObject.AddComponent<MeshCollider>();

RaycastHit hit;
Vector3 v = new Vector3();
Debug.Log(bounds.center.y + bounds.extents.y);
v.y = (bounds.center.y + bounds.extents.y);
v = transform.TransformPoint(v);
float heightWS = v.y;
v.Set(0, 0, 0);
v.y = (bounds.center.y - bounds.extents.y);
v = transform.TransformPoint(v);
float neHeightWS = v.y;
float range = heightWS - neHeightWS;
heightWS += 10; // 稍微调高一点

int index = 0;
int loopCount = 0;
while (index < count && loopCount < (count * 10))
{
    loopCount++;
    Vector3 pos = new Vector3( Random.value * bounds.extents.x * 2 - bounds.extents.x + bounds.center.x,
        0,
        Random.value * bounds.extents.z * 2 - bounds.extents.z + bounds.center.z);
    pos = transform.TransformPoint(pos);
    pos.y = heightWS;

    if (Physics.Raycast(pos, Vector3.down, out hit))
    {
        pos.y = hit.point.y;
        GrassBlade blade = new GrassBlade(pos);
        bladesArray[index++] = blade;
    }
}
```

这里用射线检测每个草的位置，计算其正确高度。

![image-20240604121806165](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240604121806165.png)

还可以调整一下，海拔越高，草地越稀疏。

![image-20240604122104986](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240604122104986.png)

如上图。计算两个绿色箭头的比值，越高的海拔生成的概率越低。

```csharp
float deltaHeight = (pos.y - neHeightWS) / range;
if (Random.value > deltaHeight)
{
	// 生草
}
```

![image-20240604122650266](/Users/remooo/Library/Application%20Support/typora-user-images/image-20240604122650266.png)

![image-20240604124602707](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604124602707.png)

当前代码链接：

- CSharp：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_PG/Assets/Scripts/GrassBlades.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_PG/Assets/Shaders/GrassBlades.shader
- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_PG/Assets/Shaders/GrassBlades.compute

现在光影啥的都没问题了。有空再加上每个叶片的弯曲。动态弯曲的效果和GS不太一样。Geometry Shader是每一帧生成一个Mesh，所以草的弯曲可以直接动态写在GS中。当前方案是，先生成好了Mesh，再传到GPU Instancing。怎么办呢？可以将不同弯曲程度的Mesh预先传到GPU Buffer中，渲染时根据弯曲程度选择关键帧就好了。还可以加一个帧插值保证流畅。

### 3. 交互草

上一节中，我们先是旋转了草的朝向，又是改变了草的倾倒。现在我们还要加上一个旋转，当一个物体靠近草，就让草朝着与物体相反的方向伏倒。这意味着又来一个旋转。这个旋转并不好设置，因此改为四元数进行。而四元数的计算在Compute Shader进行。传给材质的也是四元数，存在草片的结构体中。最后在顶点着色器中将四元数转换回仿射矩阵应用旋转。

这里再加入草的随机宽和身高。因为目前每个草Mesh都是一样的，没办法通过修改Mesh的方法修改草的高度。因此只能在Vert做顶点偏移了。

```csharp
// C#
[Range(0,0.5f)]
public float width = 0.2f;
[Range(0,1f)]
public float rd_width = 0.1f;
[Range(0,2)]
public float height = 1f;
[Range(0,1f)]
public float rd_height = 0.2f;

    GrassBlade blade = new GrassBlade(pos);
    blade.height = Random.Range(-rd_height, rd_height);
    blade.width = Random.Range(-rd_width, rd_width);
    bladesArray[index++] = blade;
// Setup 开头
GrassBlade blade = bladesBuffer[unity_InstanceID];
_HeightOffset = blade.height_offset;
_WidthOffset = blade.width_offset;
// Vert 开头
float tempHeight = v.vertex.y * _HeightOffset;
float tempWidth = v.vertex.x * _WidthOffset;
v.vertex.y += tempHeight;
v.vertex.x += tempWidth;
```

整理一下，当前的一个草Buffer存了:

```csharp
struct GrassBlade{
    public Vector3 position; // 世界坐标位置 - 需初始化
    public float height; // 草的身高偏移 - 需初始化
    public float width; // 草的宽度偏移 - 需初始化
    public float dir; // 叶片朝向 - 需初始化
    public float fade; // 随机草叶明暗 - 需初始化
    public Quaternion quaternion; // 旋转参数 - CS计算->Vert
    public float padding;

    public GrassBlade( Vector3 pos){
        position.x = pos.x;
        position.y = pos.y;
        position.z = pos.z;
        height = width = 0;
        dir = Random.Range(0, 180);
        fade = Random.Range(0.99f, 1);
        quaternion = Quaternion.identity;
        padding = 0;
    }
}
int SIZE_GRASS_BLADE = 12 * sizeof(float);
```

用来表示从向量 `v1` 旋转到向量 `v2` 的四元数 `q` ：

```glsl
float4 MapVector(float3 v1, float3 v2){
    v1 = normalize(v1);
    v2 = normalize(v2);
    float3 v = v1+v2;
    v = normalize(v);
    float4 q = 0;
    q.w = dot(v, v2);
    q.xyz = cross(v, v2);
    return q;
}
```

想要组合两个旋转的四元数，需要用乘法（注意顺序）。

假设有两个四元数 $$q_1=a+b i+c j+d k$$ 和 $$q_2=w+x i+y j+ z k$$ 。它们的乘积 $$q_1 q_2$$ 计算公式是 :
$$
q_1 q_2=\\(a w-b x-c y-d z)+\\(a x+b w+c z-d y) i+\\(a y- 
b z+c w+d x) j+\\(a z+b y-c x+d w) k
$$

其中 $$a, b, c, d$$ 是 $$q_1$$ 的实部和虚部分量, $$w, x, y, z$$ 是 $$q_2$$​的实部和虚部分量。

```glsl
float4 quatMultiply(float4 q1, float4 q2) {
    // q1 = a + bi + cj + dk
    // q2 = x + yi + zj + wk
    // Result = q1 * q2
    return float4(
        q1.w * q2.x + q1.x * q2.w + q1.y * q2.z - q1.z * q2.y, // X component
        q1.w * q2.y - q1.x * q2.z + q1.y * q2.w + q1.z * q2.x, // Y component
        q1.w * q2.z + q1.x * q2.y - q1.y * q2.x + q1.z * q2.w, // Z component
        q1.w * q2.w - q1.x * q2.x - q1.y * q2.y - q1.z * q2.z  // W (real) component
    );
}
```

要确定草是往哪个地方倒，就需要获取交互物体trampler的Pos，也就是其Transform组件。并且每一帧都通过`SetVector`传到GPU Buffer中，给Compute Shader用，所以把GPU的内存地址当作ID存着，不需要每次都用字符串访问。还要确定多大范围内的草要倒下，倒与不倒之间怎么过渡，给GPU传一个 `trampleRadius` ，由于这个是常数，就不用每一帧都修改，因此直接用字符串Set一下就好了。

```csharp
// CSharp
public Transform trampler;
[Range(0.1f,5f)]
public float trampleRadius = 3f;
...
Init(){
    shader.SetFloat("trampleRadius", trampleRadius);
    tramplePosID = Shader.PropertyToID("tramplePos");
}
Update(){
    shader.SetVector(tramplePosID, pos);
}
```

本节把所有旋转的操作都丢进Compute Shader里面一次算完，直接返回一个四元数给材质。首先是`q1`计算随机朝向的四元数，`q2`计算随机倾倒，`qt`计算交互的倾倒。这里可以在Inspector开放一个交互的系数。

```glsl
[numthreads(THREADGROUPSIZE,1,1)]
void BendGrass (uint3 id : SV_DispatchThreadID)
{
    GrassBlade blade = bladesBuffer[id.x];

    float3 relativePosition = blade.position - tramplePos.xyz;
    float dist = length(relativePosition);

    float4 qt;
    
    if (dist<trampleRadius){
        float eff = ((trampleRadius - dist)/trampleRadius) * 0.6;
        qt = MapVector(float3(0,1,0), float3(relativePosition.x*eff,1,relativePosition.y*eff));
    }else{
        qt = MapVector(float3(0,1,0),float3(0,1,0));
    }
    
    float2 offset = (blade.position.xz + wind.xy * time * wind.z) * wind.w;
    float noise = perlin(offset.x, offset.y) * 2 - 1;
    noise *= maxBend;
    float4 q1 = MapVector(float3(0,1,0), (float3(wind.x * noise,1,wind.y*noise)));
    float faceTheta = blade.dir * 3.1415f / 180.0f;
    float4 q2 = MapVector(float3(1,0,0),float3(cos(faceTheta),0,sin(faceTheta)));
    blade.quaternion = quatMultiply(qt,quatMultiply(q2,q1));
    bladesBuffer[id.x] = blade;
}
```

然后四元数到旋转矩阵的方法是：

```glsl
float4x4 quaternion_to_matrix(float4 quat)
{
    float4x4 m = float4x4(float4(0, 0, 0, 0), float4(0, 0, 0, 0), float4(0, 0, 0, 0), float4(0, 0, 0, 0));

    float x = quat.x, y = quat.y, z = quat.z, w = quat.w;
    float x2 = x + x, y2 = y + y, z2 = z + z;
    float xx = x * x2, xy = x * y2, xz = x * z2;
    float yy = y * y2, yz = y * z2, zz = z * z2;
    float wx = w * x2, wy = w * y2, wz = w * z2;

    m[0][0] = 1.0 - (yy + zz);
    m[0][1] = xy - wz;
    m[0][2] = xz + wy;

    m[1][0] = xy + wz;
    m[1][1] = 1.0 - (xx + zz);
    m[1][2] = yz - wx;

    m[2][0] = xz - wy;
    m[2][1] = yz + wx;
    m[2][2] = 1.0 - (xx + yy);

    m[0][3] = _Position.x;
    m[1][3] = _Position.y;
    m[2][3] = _Position.z;
    m[3][3] = 1.0;

    return m;
}
```

然后应用一下。

```glsl
void vert(inout appdata_full v, out Input data)
{
    UNITY_INITIALIZE_OUTPUT(Input, data);

    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    float tempHeight = v.vertex.y * _HeightOffset;
    float tempWidth = v.vertex.x * _WidthOffset;
    v.vertex.y += tempHeight;
    v.vertex.x += tempWidth;
    // 应用模型顶点变换
    v.vertex = mul(_Matrix, v.vertex);
    v.vertex.xyz += _Position;
    // 计算逆转置矩阵用于法线变换
    v.normal = mul((float3x3)transpose(_Matrix), v.normal);
    #endif
}

void setup()
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
        // 获取Compute Shader计算结果
        GrassBlade blade = bladesBuffer[unity_InstanceID];
        _HeightOffset = blade.height_offset;
        _WidthOffset = blade.width_offset;
        _Fade = blade.fade; // 设置明暗
        _Matrix = quaternion_to_matrix(blade.quaternion); // 设置最终转转矩阵  
        _Position = blade.position; // 设置位置
    #endif
}
```

![image-20240604191451278](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604191451278.png)

![image-20240604171102956](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604171102956.png)

当前代码链接：

- CSharp：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_Finally/Assets/Scripts/GrassBlades.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_Finally/Assets/Shaders/GrassBlades.shader
- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L5_Finally/Assets/Shaders/GrassBlades.compute



## 4. 总结/小测试

How do you programmatically get the thread group sizes of a kernel?

![image-20240604125326611](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604125326611.png)

When defining a Mesh in code, the number of normals must be the same as the number of vertex positions. True or false.

![image-20240604125451443](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240604125451443.png)

