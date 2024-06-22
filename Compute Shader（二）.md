# Compute Shader学习笔记（二）

[TOC]

## 前言

上一篇文章中，初步认识了Compute Shader，实现一些简单的效果。所有的代码都在：https://github.com/Remyuu/Unity-Compute-Shader-Learn 。main分支是初始代码，可以下载完整的工程跟着我敲一遍。PS：每一个版本的代码我都单独开了分支。

![image-20240525213949959](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525213949959.png)

这一篇文章学习如何使用Compute Shader制作：

- 后处理效果
- 粒子系统

上一篇文章没有提及GPU的架构，是因为我觉得一上来就解释一大堆名词根本听不懂QAQ。有了实际编写Compute Shader的经验，就可以将抽象的概念和实际的代码联系起来。

CUDA在GPU上的**执行程序**可以用三层架构来说明：

Grid - 对应一个Kernel
|-Block - 一个Grid有多个Block，执行相同的程序
| |-Thread - GPU上最基本的运算单元

![image-20240527110848647](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527110848647.png)

Thread是GPU最基础的单元，不同Thread中自然就会有信息交换。为了有效地支持大量并行线程的运行，并解决这些线程之间的数据交换需求，内存被设计成多个层次。因此**存储角度**也可以分为三层：

Per-Thread memory - 一个Thread内，传输周期是一个时钟周期（小于1纳秒），速度可以比全局内存快几百倍。
Shared memory - 一个Block之间，速度比全局快很多。
Global memory - 所有线程之间，但速度最慢，通常是GPU的瓶颈。Volta架构使用了HBM2作为设备的全局内存，Turing则是用了GDDR6。

如果超过内存大小限制，则会被推到容量更大但是更慢的存储空间上。

Shared Memory和L1 cache共享同一个物理空间，但是功能上有区别：前者需要手动管理，后者由硬件自动管理。我的理解是，Shared Memory 功能上类似于一个可编程的L1缓存。



![image-20240527111302167](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527111302167.png)

在NVIDIA的CUDA架构中，**流式多处理器（Streaming Multiprocessor, SM）**是GPU上的一个处理单元，负责执行分配给它的**线程块（Blocks）**中的线程。**流处理器（Stream Processors）**，也称为“CUDA核心”，是SM内的处理元件，每个流处理器可以并行处理多个线程。总的来说：

- **GPU -> Multi-Processors (SMs) -> Stream Processors**

即，GPU包含多个SM（也就是多处理器），每个SM包含多个流处理器。每个流处理器负责执行一个或多个线程（Thread）的计算指令。

在GPU中，**Thread（线程）**是执行计算的最小单元，**Warp（纬度）**是CUDA中的基本执行单位。

在NVIDIA的CUDA架构中，每个**Warp**通常包含32个**线程**（AMD有64个）。**Block（块）**是一个线程组，包含多个线程。在CUDA中，一个**Block**可以包含多个**Warp**。**Kernel（内核）**是在GPU上执行的一个函数，你可以将其视为一段特定的代码，这段代码被所有激活的线程并行执行。总的来说：

- **Kernel -> Grid -> Blocks -> Warps -> Threads**

但在日常开发中，通常需要同时执行的**线程（Threads）**远超过32个。

为了解决软件需求与硬件架构之间的数量不匹配问题，GPU采用了一种策略：将属于同一个块（Block）的线程分组。这种分组被称为“Warp”，每个Warp包含固定数量的线程。当需要执行的线程数量超过一个Warp所能包含的数量时，GPU会调度额外的Warp。这样做的原则是确保没有任何线程被遗漏，即便这意味着需要启动更多的Warp。

举个例子，如果一个块（Block）有128个线程（Thread），并且我的显卡身穿皮夹克（Nvidia每个Warp有32个Thread），那么一个块（Block）就会有 `128/32=4` 个Warp。举一个极端的例子，如果有129个线程，那么就会开5个Warp。有31个线程位置将直接空闲！因此我们在写Compute Shader时，`[numthreads(a,b,c)]` 中的 `a*b*c` 最好是32的倍数，减少CUDA核心的浪费。

读到这里，想必你一定会很混乱。我按照个人的理解画了个图。若有错误请指出。

![image-20240527132257443](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527132257443.png)

## L3 后处理效果

> 当前构建基于BIRP管线，SRP管线只需要修改几处代码。

![image-20240527195146403](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527195146403.png)

这一章关键在于构建一个抽象基类管理Compute Shader所需的资源（第一节）。然后基于这个抽象基类，编写一些简单的后处理效果，比如高斯模糊、灰阶效果、低分辨率像素效果以及夜视仪效果等等。这一章的知识点的小总结：

- 获取和处理Camera的渲染贴图

- `ExecuteInEditMode` 关键词

- `SystemInfo.supportsComputeShaders` 检查系统是否支持

- `Graphics.Blit()` 函数的使用

- 用 `smoothstep()` 制作各种效果

- 多个Kernel之间传输数据 `Shared` 关键词

### 1. 介绍与准备工作

后处理效果需要准备两张贴图，一个只读，另一个可读写。至于贴图从哪来，都说是后处理了，那肯定从相机身上获取贴图，也就是Camera组件上的Target Texture。

- Source：只读
- Destination：可读写，用于最终输出

![image-20240523135324863](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523135324863.png)

由于后续会实现多种后处理效果，因此抽象出一个基类，减少后期工作量。

在基类中封装以下特性：

- 初始化资源（创建贴图、Buffer等）
- 管理资源（比方说屏幕分辨率改变后，重新创建Buffer等等）
- 硬件检查（检查当前设备是否支持Compute Shader）

抽象类完整代码链接：https://pastebin.com/9pYvHHsh

首先，当脚本实例被激活或者附加到活着的GO的时候，调用 `OnEnable()` 。在里面写初始化的操作。检查硬件是否支持、检查Compute Shader是否在Inspector上绑定、获取指定的Kernel、获取当前GO的Camera组件、创建纹理以及设置初始化状态为真。

```csharp
if (!SystemInfo.supportsComputeShaders)
	...

if (!shader)
	...

kernelHandle = shader.FindKernel(kernelName);

thisCamera = GetComponent<Camera>();

if (!thisCamera)
	...

CreateTextures();

init = true;
```

创建两个纹理 `CreateTextures()` ，一个Source一个Destination，尺寸为摄像机分辨率。

```csharp
texSize.x = thisCamera.pixelWidth;
texSize.y = thisCamera.pixelHeight;

if (shader)
{
    uint x, y;
    shader.GetKernelThreadGroupSizes(kernelHandle, out x, out y, out _);
    groupSize.x = Mathf.CeilToInt((float)texSize.x / (float)x);
    groupSize.y = Mathf.CeilToInt((float)texSize.y / (float)y);
}

CreateTexture(ref output);
CreateTexture(ref renderedSource);

shader.SetTexture(kernelHandle, "source", renderedSource);
shader.SetTexture(kernelHandle, "outputrt", output);
```

具体纹理的创建：

```csharp
protected void CreateTexture(ref RenderTexture textureToMake, int divide=1)
{
    textureToMake = new RenderTexture(texSize.x/divide, texSize.y/divide, 0);
    textureToMake.enableRandomWrite = true;
    textureToMake.Create();
}
```

这样就完成初始化了，当摄像机完成场景渲染并准备显示到屏幕上时，Unity会调用 `OnRenderImage()` ，这个时候就开始调用Compute Shader开始计算了。若当前没初始化好或者没shader，就Blit一下，把source直接拷给destination，即啥也不干。 `CheckResolution(out _)`  这个方法检查渲染纹理的分辨率是否需要更新，如果要，就重新生成一下Texture。完事之后，就到了老生常谈的Dispatch阶段啦。这里就需要将source贴图通过Buffer传给GPU，计算完毕后，传回给destination。

```csharp
protected virtual void OnRenderImage(RenderTexture source, RenderTexture destination)
{
    if (!init || shader == null)
    {
        Graphics.Blit(source, destination);
    }
    else
    {
        CheckResolution(out _);
        DispatchWithSource(ref source, ref destination);
    }
}
```

注意看，这里我们没有用什么 `SetData()` 或者是 `GetData()` 之类的操作。因为现在所有数据都在GPU上，我们直接命令GPU自产自销就好了，CPU不要趟这滩浑水。如果将纹理取回内存，再传给GPU，性能就相当糟糕。

```csharp
protected virtual void DispatchWithSource(ref RenderTexture source, ref RenderTexture destination)
{
    Graphics.Blit(source, renderedSource);

    shader.Dispatch(kernelHandle, groupSize.x, groupSize.y, 1);

    Graphics.Blit(output, destination);
}
```

我不信邪，非得传回CPU再传回GPU，测试结果相当震惊，性能竟然差了4倍以上。因此我们需要减少CPU和GPU之间的通信，这是使用Compute Shader时非常需要关心的。

```csharp
// 笨蛋方法
protected virtual void DispatchWithSource(ref RenderTexture source, ref RenderTexture destination)
{
    // 将源贴图Blit到用于处理的贴图
    Graphics.Blit(source, renderedSource);

    // 使用计算着色器处理贴图
    shader.Dispatch(kernelHandle, groupSize.x, groupSize.y, 1);

    // 将输出贴图复制到一个Texture2D对象中，以便读取数据到CPU
    Texture2D tempTexture = new Texture2D(renderedSource.width, renderedSource.height, TextureFormat.RGBA32, false);
    RenderTexture.active = output;
    tempTexture.ReadPixels(new Rect(0, 0, output.width, output.height), 0, 0);
    tempTexture.Apply();
    RenderTexture.active = null;

    // 将Texture2D数据传回GPU到一个新的RenderTexture
    RenderTexture tempRenderTexture = RenderTexture.GetTemporary(output.width, output.height);
    Graphics.Blit(tempTexture, tempRenderTexture);

    // 最终将处理后的贴图Blit到目标贴图
    Graphics.Blit(tempRenderTexture, destination);

    // 清理资源
    RenderTexture.ReleaseTemporary(tempRenderTexture);
    Destroy(tempTexture);
}
```

![image-20240523150247043](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523150247043.png)

接下来开始编写第一个后处理效果。

#### 小插曲：奇怪的BUG

另外插播一个奇怪bug。

在Compute Shader中，如果最终输出的贴图结果名字是output，那么在某些API比如Metal中，就会出问题。解决方法是，改个名字。

```glsl
RWTexture2D<float4> outputrt;
```

![image-20240523193113402](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523193113402.png)

### 2. RingHighlight效果

![image-20240523151347009](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523151347009.png)

创建RingHighlight类，继承自刚刚编写的基类。

![image-20240523150920025](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523150920025.png)

重载初始化方法，指定Kernel。

```csharp
protected override void Init()
{
    center = new Vector4();
    kernelName = "Highlight";
    base.Init();
}
```

重载渲染方法。想要实现聚焦某个角色的效果，则需要给Compute Shader传入角色的屏幕空间的坐标 `center` 。并且，如果在Dispatch之前，屏幕分辨率发生改变，那么重新初始化。

```csharp
protected void SetProperties()
{
    float rad = (radius / 100.0f) * texSize.y;
    shader.SetFloat("radius", rad);
    shader.SetFloat("edgeWidth", rad * softenEdge / 100.0f);
    shader.SetFloat("shade", shade);
}

protected override void OnRenderImage(RenderTexture source, RenderTexture destination)
{
    if (!init || shader == null)
    {
        Graphics.Blit(source, destination);
    }
    else
    {
        if (trackedObject && thisCamera)
        {
            Vector3 pos = thisCamera.WorldToScreenPoint(trackedObject.position);
            center.x = pos.x;
            center.y = pos.y;
            shader.SetVector("center", center);
        }
        bool resChange = false;
        CheckResolution(out resChange);
        if (resChange) SetProperties();
        DispatchWithSource(ref source, ref destination);
    }
}
```

并且改变Inspector面板的时候可以实时看到参数变化效果，添加 `OnValidate()` 方法。

```csharp
private void OnValidate()
{
    if(!init)
        Init();

    SetProperties();
}
```

GPU中，该怎么制作一个圆内没有阴影，圆的边缘平滑过渡，过渡层外是阴影的效果呢？基于上一篇文章判断一个点是否在圆内的方法，我们用 `smoothstep()` ，处理过渡层即可。

```glsl
#pragma kernel Highlight

Texture2D<float4> source;
RWTexture2D<float4> outputrt;
float radius;
float edgeWidth;
float shade;
float4 center;

float inCircle( float2 pt, float2 center, float radius, float edgeWidth ){
    float len = length(pt - center);
    return 1.0 - smoothstep(radius-edgeWidth, radius, len);
}

[numthreads(8, 8, 1)]
void Highlight(uint3 id : SV_DispatchThreadID)
{
    float4 srcColor = source[id.xy];
    float4 shadedSrcColor = srcColor * shade;
    float highlight = inCircle( (float2)id.xy, center.xy, radius, edgeWidth);
    float4 color = lerp( shadedSrcColor, srcColor, highlight );

    outputrt[id.xy] = color;

}
```



![image-20240523152439858](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523152439858.png)

当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_RingHighlight/Assets/Shaders/RingHighlight.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_RingHighlight/Assets/Scripts/RingHighlight.cs

### 3. 模糊效果

![image-20240523161258210](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523161258210.png)

模糊效果原理很简单，每一个像素采样周边的 `n*n` 个像素加权平均就可以得到最终效果。



但是有效率问题。众所周知，减少对纹理的采样次数对优化非常重要。如果每个像素都需要采样`20*20`个周边像素，那么渲染一个像素就需要采样`400`次，显然是无法接受的。并且，对于单个像素而言，采集周边一整个矩形像素的操作在Compute Shader中很难处理。怎么解决呢？



通常做法是，横着采样一遍，再竖着采样一遍。什么意思呢？对于每一个像素，只在x方向上采样20个像素，y方向上采样20个像素，总共采样`20+20`个像素，再加权平均。这种方法不仅减少了采样次数，还更符合Compute Shader的逻辑。横着采样，设置一个Kernel；竖着采样，设置另一个Kernel。

```glsl
#pragma kernel HorzPass
#pragma kernel Highlight
```

由于Dispatch是顺序执行的，因此我们计算完水平的模糊后，利用计算好的结果再垂直采样一遍。

```csharp
shader.Dispatch(kernelHorzPassID, groupSize.x, groupSize.y, 1);
shader.Dispatch(kernelHandle, groupSize.x, groupSize.y, 1);
```

做完模糊操作之后，再结合上一节的RingHighlight，完工！

有一点不同的是，再计算完水平模糊后，怎么将结果传给下一个Kernel呢？答案呼之欲出了，直接使用 `shared` 关键词。具体步骤如下。

CPU中声明存储水平模糊纹理的引用，制作水平纹理的kernel，并绑定。

```csharp
RenderTexture horzOutput = null;
int kernelHorzPassID;

protected override void Init()
{
	...
    kernelHorzPassID = shader.FindKernel("HorzPass");
	...
}
```

还需要额外在GPU中开辟空间，用来存储第一个kernel的结果。

```csharp
protected override void CreateTextures()
{
    base.CreateTextures();
    shader.SetTexture(kernelHorzPassID, "source", renderedSource);

    CreateTexture(ref horzOutput);

    shader.SetTexture(kernelHorzPassID, "horzOutput", horzOutput);
    shader.SetTexture(kernelHandle, "horzOutput", horzOutput);
}
```

GPU上这样设置：

```glsl
shared Texture2D<float4> source;
shared RWTexture2D<float4> horzOutput;
RWTexture2D<float4> outputrt;
```

另外有个疑问， `shared` 这个关键词好像加不加都一样，实际测试不同的kernel都可以访问到。那请问shared还有什么意义呢？

在Unity中，变量前加`shared`表示这个资源不是每次调用都重新初始化，而是保持其状态，供不同的shader或dispatch调用使用。这有助于在不同的shader调用之间共享数据。标记了 `shared` 可以帮助编译器优化出更高性能的代码。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525212654641.png" alt="image-20240525212654641" style="zoom:50%;" />

在计算边界的像素时，会遇到可用像素数量不足的情况。要么就是左边剩下的像素不足 `blurRadius` ，要么右边剩余像素不足。因此先算出安全的左索引，然后再计算从左到右最大可以取多少。

```glsl
[numthreads(8, 8, 1)]
void HorzPass(uint3 id : SV_DispatchThreadID)
{
    int left = max(0, (int)id.x-blurRadius);
    int count = min(blurRadius, (int)id.x) + min(blurRadius, source.Length.x - (int)id.x);
    float4 color = 0;

    uint2 index = uint2((uint)left, id.y);

    [unroll(100)]
    for(int x=0; x<count; x++){
        color += source[index];
        index.x++;
    }

    color /= (float)count;
    horzOutput[id.xy] = color;

}

[numthreads(8, 8, 1)]
void Highlight(uint3 id : SV_DispatchThreadID)
{
    //Vert blur
    int top = max(0, (int)id.y-blurRadius);
    int count = min(blurRadius, (int)id.y) + min(blurRadius, source.Length.y - (int)id.y);
    float4 blurColor = 0;

    uint2 index = uint2(id.x, (uint)top);

    [unroll(100)]
    for(int y=0; y<count; y++){
        blurColor += horzOutput[index];
        index.y++;
    }

    blurColor /= (float)count;

    float4 srcColor = source[id.xy];
    float4 shadedBlurColor = blurColor * shade;
    float highlight = inCircle( (float2)id.xy, center.xy, radius, edgeWidth);
    float4 color = lerp( shadedBlurColor, srcColor, highlight );

    outputrt[id.xy] = color;

}
```



当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_BlurEffect/Assets/Shaders/BlurHighlight.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_BlurEffect/Assets/Scripts/BlurHighlight.cs

### 4. 高斯模糊

和上面不同的是，采样之后不再是取平均值，而是用一个高斯函数加权求得。
$$
G(x)=\frac{1}{\sqrt{2 \pi \sigma^2}} e^{-\frac{x^2}{2 \sigma^2}}
$$

其中， $$\sigma$$​ 是标准差，控制宽度。

有关更多Blur的内容：https://www.gamedeveloper.com/programming/four-tricks-for-fast-blurring-in-software-and-hardware#close-modal

由于这个计算量还有不小的，如果每一个像素都去计算一次这个式子就非常耗。我们用预计算的方式，将计算结果通过Buffer的方式传到GPU上。由于两个kernel都需要使用，在Buffer声明的时候加一个shared。

```csharp
float[] SetWeightsArray(int radius, float sigma)
{
    int total = radius * 2 + 1;
    float[] weights = new float[total];
    float sum = 0.0f;

    for (int n=0; n<radius; n++)
    {
        float weight = 0.39894f * Mathf.Exp(-0.5f * n * n / (sigma * sigma)) / sigma;
        weights[radius + n] = weight;
        weights[radius - n] = weight;
        if (n != 0)
            sum += weight * 2.0f;
        else
            sum += weight;
    }
    // normalize kernels
    for (int i=0; i<total; i++) weights[i] /= sum;

    return weights;
}

private void UpdateWeightsBuffer()
{
    if (weightsBuffer != null)
        weightsBuffer.Dispose();

    float sigma = (float)blurRadius / 1.5f;

    weightsBuffer = new ComputeBuffer(blurRadius * 2 + 1, sizeof(float));
    float[] blurWeights = SetWeightsArray(blurRadius, sigma);
    weightsBuffer.SetData(blurWeights);

    shader.SetBuffer(kernelHorzPassID, "weights", weightsBuffer);
    shader.SetBuffer(kernelHandle, "weights", weightsBuffer);
}
```

![image-20240523192415194](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523192415194.png)

完整代码：

- https://pastebin.com/0qWtUKgy
- https://pastebin.com/A6mDKyJE



### 5. 低分辨率效果



> GPU：真是酣畅淋漓的计算啊。



![image-20240523194250473](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523194250473.png)



让一张高清的纹理边模糊，同时不修改分辨率。实现方法很简单，每 `n*n` 个像素，都只取左下角的像素颜色即可。利用整数的特性，id.x索引先除n，再乘上n就可以了。



```glsl
uint2 index = (uint2(id.x, id.y)/3) * 3;
float3 srcColor = source[index].rgb;
float3 finalColor = srcColor;
```



效果已经放在上面了。但是这个效果太锐利了，通过添加噪声，柔化锯齿。



```glsl
uint2 index = (uint2(id.x, id.y)/3) * 3;

float noise = random(id.xy, time);
float3 srcColor = lerp(source[id.xy].rgb, source[index],noise);
float3 finalColor = srcColor;
```



![image-20240523195729945](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523195729945.png)



每 `n*n` 个格子的像素不在只取左下角的颜色，而是取原本颜色和左下角颜色的随机插值结果。效果一下子就精细了不少。当n比较大的时候，还能看到下面这样的效果。只能说不太好看，但是在一些故障风格道路中还是可以继续探索。



![image-20240523200042600](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523200042600.png)



如果想要得到噪声感的画面，可以尝试lerp的两端添加系数，比如：

```glsl
float3 srcColor = lerp(source[id.xy].rgb * 2, source[index],noise);
```

![image-20240523204538679](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523204538679.png)



### 6. 灰阶效果与染色

> Grayscale Effect & Tinted

将彩色图像转换为灰阶图像的过程涉及将每个像素的RGB值转换为一个单一的颜色值。这个颜色值是RGB值的加权平均值。这里有两种方法，一种是简单平均，一种是符合人眼感知的加权平均。

1. 平均值法（简单但不准确）：
$$
\text { Gray }=\frac{R+G+B}{3}
$$

这种方法对所有颜色通道给予相同的权重。
2. 加权平均法（更准确, 反映人眼感知）：
$$
\text { Gray }=0.299 \times R+0.587 \times G+0.114 \times B
$$

这种方法根据人眼对绿色更敏感、对红色次之、对蓝色最不敏感的特点, 给予不同颜色通道不同的权重。（下面的截图效果不太好，我也没看出来lol）

![image-20240523203727289](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523203727289.png)

加权后，再简单地颜色混合（乘法），最后lerp得到可控的染色强度结果。

```glsl
uint2 index = (uint2(id.x, id.y)/6) * 6;

float noise = random(id.xy, time);
float3 srcColor = lerp(source[id.xy].rgb, source[index],noise);
// float3 finalColor = srcColor;

float3 grayScale = (srcColor.r+srcColor.g+srcColor.b)/3.0;
// float3 grayScale = srcColor.r*0.299f+srcColor.g*0.587f+srcColor.b*0.114f;

float3 tinted = grayScale * tintColor.rgb;
float3 finalColor = lerp(srcColor, tinted, tintStrength);

outputrt[id.xy] = float4(finalColor, 1);
```

染一个废土颜色：

![image-20240523204103050](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523204103050.png)

### 7. 屏幕扫描线效果

首先 `uvY` 将坐标归一化到 [0,1] 。

`lines` 是控制扫描线数量的一个参数。

然后增加一个时间偏移，系数控制偏移速度。可以开放一个参数控制线条偏移的速度。

```glsl
float uvY = (float)id.y/(float)source.Length.y;
float scanline = saturate(frac(uvY * lines + time * 3));
```

![image-20240523212252649](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523212252649.png)

这个“线”看起来不太够“线”，减个肥。

```glsl
float uvY = (float)id.y/(float)source.Length.y;
float scanline = saturate(smoothstep(0.1,0.2,frac(uvY * lines + time * 3)));
```

![image-20240523212326080](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523212326080.png)

然后lerp上颜色。

```glsl
float uvY = (float)id.y/(float)source.Length.y;
float scanline = saturate(smoothstep(0.1, 0.2, frac(uvY * lines + time*3)) + 0.3);
finalColor = lerp(source[id.xy].rgb*0.5, finalColor, scanline);
```



![image-20240523212123495](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523212123495.png)

“减肥”前后，各取所需吧！

![image-20240523205916053](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240523205916053.png)

### 8. 夜视仪效果

这一节总结上面所有内容，实现一个夜视仪的效果。先做一个单眼效果。

```glsl
float2 pt = (float2)id.xy;
float2 center = (float2)(source.Length >> 1);
float inVision = inCircle(pt, center, radius, edgeWidth);
float3 blackColor = float3(0,0,0);
finalColor = lerp(blackColor, finalColor, inVision);
```

![image-20240524123414363](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240524123414363.png)

双眼效果不同点在于有两个圆心，计算得到的两个遮罩vision用 `max()` 或者是 `saturate()` 合并即可。

```glsl
float2 pt = (float2)id.xy;
float2 centerLeft = float2(source.Length.x / 3.0, source.Length.y /2);
float2 centerRight = float2(source.Length.x / 3.0 * 2.0, source.Length.y /2);
float inVisionLeft = inCircle(pt, centerLeft, radius, edgeWidth);
float inVisionRight = inCircle(pt, centerRight, radius, edgeWidth);
float3 blackColor = float3(0,0,0);
// float inVision = max(inVisionLeft, inVisionRight);
float inVision = saturate(inVisionLeft + inVisionRight);
finalColor = lerp(blackColor, finalColor, inVision);
```



![image-20240524125012959](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240524125012959.png)



当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_NightVision/Assets/Shaders/NightVision.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_NightVision/Assets/Scripts/NightVision.cs



### 9. 平缓过渡线条

思考一下，我们应该怎么在屏幕上画一条平滑过渡的直线。



![image-20240524224714449](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240524224714449.png)



`smoothstep()` 函数可以完成这个操作，熟悉这个函数的读者可以略过这一段。这个函数用来创建平滑的渐变。`smoothstep(edge0, edge1, x)` 函数在`x`在 `edge0` 和 `edge1` 之间时，输出值从0渐变到1。如果 `x < edge0` ，返回0；如果 `x > edge1` ，返回1。其输出值是根据Hermite插值计算的：
$$
S(t) = 3t^2 - 2t^3 \\
t = \frac{x-edge0}{edge1-edge0}
$$


![image-20240524230108142](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240524230108142.png)

```glsl
float onLine(float position, float center, float lineWidth, float edgeWidth) {
    float halfWidth = lineWidth / 2.0;
    float edge0 = center - halfWidth - edgeWidth;
    float edge1 = center - halfWidth;
    float edge2 = center + halfWidth;
    float edge3 = center + halfWidth + edgeWidth;

    return smoothstep(edge0, edge1, position) - smoothstep(edge2, edge3, position);
}
```

上面代码中，传入的参数都已经归一化 `[0,1]`。position 是考察的点的位置，center 是线的中心位置，lineWidth 是线的实际宽度，edgeWidth 是边缘的宽度，用于平滑过渡。我实在对我的表达能力感到不悦！至于怎么算的，我给大家画个图理解吧！

大概就是：$$1-0=0$$，$$1-0.5=0.5$$ ，$$1-1=0$$。

![image-20240524231704668](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240524231704668.png)

思考一下，怎么画一个平滑过渡的圆。

对于每个点，先计算与圆心的距离向量，结果返回给 `position` ，并且计算其长度返回给 `len` 。

模仿上面两个 `smoothstep` 做差的方法，通过减去外边缘插值结果来生成一个环形的线条效果。

```glsl
float circle(float2 position, float2 center, float radius, float lineWidth, float edgeWidth){
    position -= center;
    float len = length(position);
    //Change true to false to soften the edge
    float result = smoothstep(radius - lineWidth / 2.0 - edgeWidth, radius - lineWidth / 2.0, len) - smoothstep(radius + lineWidth / 2.0, radius + lineWidth / 2.0 + edgeWidth, len);

    return result;
}
```

![image-20240525194658894](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525194658894.png)



### 10. 扫描线效果

然后一条横线、一条竖线，套娃几个圆，做一个雷达扫描的效果。

```glsl
float3 color = float3(0.0f,0.0f,0.0f);
color += onLine(uv.y, center.y, 0.002, 0.001) * axisColor.rgb;//xAxis
color += onLine(uv.x, center.x, 0.002, 0.001) * axisColor.rgb;//yAxis
color += circle(uv, center, 0.2f, 0.002, 0.001) * axisColor.rgb;
color += circle(uv, center, 0.3f, 0.002, 0.001) * axisColor.rgb;
color += circle(uv, center, 0.4f, 0.002, 0.001) * axisColor.rgb;
```

再画一个扫描线，并且带有轨迹。

```glsl
float sweep(float2 position, float2 center, float radius, float lineWidth, float edgeWidth) {
    float2 direction = position - center;
    float theta = time + 6.3;
    float2 circlePoint = float2(cos(theta), -sin(theta)) * radius;
    float projection = clamp(dot(direction, circlePoint) / dot(circlePoint, circlePoint), 0.0, 1.0);
    float lineDistance = length(direction - circlePoint * projection);

    float gradient = 0.0;
    const float maxGradientAngle = PI * 0.5;

    if (length(direction) < radius) {
        float angle = fmod(theta + atan2(direction.y, direction.x), PI2);
        gradient = clamp(maxGradientAngle - angle, 0.0, maxGradientAngle) / maxGradientAngle * 0.5;
    }

    return gradient + 1.0 - smoothstep(lineWidth, lineWidth + edgeWidth, lineDistance);
}
```

添加到颜色中。

```glsl
...
color += sweep(uv, center, 0.45f, 0.003, 0.001) * sweepColor.rgb;
...
```

![image-20240525201012253](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525201012253.png)

当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_HUDOverlay/Assets/Shaders/HUDOverlay.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L3_HUDOverlay/Assets/Scripts/HUDOverlay.cs



### 11. 渐变背景阴影效果

这个效果可以用在字幕或者是一些说明性文字之下。虽然可以直接在UI Canvas中加一张贴图，但是使用Compute Shader可以实现更加灵活的效果以及资源的优化。

![image-20240525223106269](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525223106269.png)

字幕、对话文字背景一般都在屏幕下方，上方不作处理。同时需要较高的对比度，因此对原有画面做一个灰度处理、并且指定一个阴影。

```glsl
if (id.y<(uint)tintHeight){
    float3 grayScale = (srcColor.r + srcColor.g + srcColor.b) * 0.33 * tintColor.rgb;
    float3 shaded = lerp(srcColor.rgb, grayScale, tintStrength) * shade;
    ... // 接下文
}else{
    color = srcColor;
}
```

![image-20240526123814179](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240526123814179.png)

渐变效果。

```glsl
	...// 接上文
	float srcAmount = smoothstep(tintHeight-edgeWidth, (float)tintHeight, (float)id.y);
	...// 接下文
```

![image-20240526123950425](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240526123950425.png)

最后再lerp起来。

```glsl
	...// 接上文
    color = lerp(float4(shaded, 1), srcColor, srcAmount);
```

![image-20240525223024595](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525223024595.png)



### 12. 总结/小测试



If `id.xy = [ 100, 30 ]`. What would be the return value of

`inCircle((float2)id.xy, float2(130, 40), 40, 0.1)`

![image-20240525221133085](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221133085.png)



When creating a blur effect which answer describes our approach best?

![image-20240525221153986](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221153986.png)



Which answer would create a blocky low resolution version of the source image?

![image-20240525221211250](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221211250.png)



What is

`smoothstep(5, 10, 6)`; ?

![image-20240525221231538](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221231538.png)



If a and b are both vectors. Which answer best describes

`dot(a,b)/dot(b,b);` ?

![image-20240525221246055](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221246055.png)



What is _MainTex_TexelSize.x? If _MainTex is 512 x 256 pixel resolution.

![image-20240525221310143](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240525221310143.png)



### 13. 利用Blit结合Material做后处理

除了使用Compute Shader制作后处理，还有一种简单的方法。

```glsl
// .cs
Graphics.Blit(source, dest, material, passIndex);
// .shader
Pass{
    CGPROGRAM
    #pragma vertex vert_img
    #pragma fragment frag
    fixed4 frag(v2f_img input) : SV_Target{
        return tex2D(_MainTex, input.uv);
    }
    ENDCG
}
```

通过结合Shader来处理图像数据。

那么问题来了，两者有什么区别？而且传进来的不是一张纹理吗，哪来的顶点？

答：

第一个问题。这种方法称为“屏幕空间着色”，完全集成在Unity的图形管线中，性能其实比Compute Shader更高。而Compute Shader提供了对GPU资源的更细粒度控制。它不受图形管线的限制，可以直接访问和修改纹理、缓冲区等资源。

第二个问题。注意看 `vert_img` 。在UnityCG中可以找到如下定义：

![image-20240526132022150](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240526132022150.png)

![image-20240526132046631](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240526132046631.png)

Unity会自动将传进来的纹理自动转换为两个三角形（一个充满屏幕的矩形），我们用材质的方法编写后处理时直接在frag上写就好了。



## L4 粒子效果与群集行为模拟

![image-20240528210234250](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528210234250.png)

本章节使用Compute Shader生成粒子。学习如何使用`DrawProcedural`和`DrawMeshInstancedIndirect`，也就是GPU Instancing。



知识点总结：

- Compute Shader、Material、C#脚本和Shader共同协作
- `Graphics.DrawProcedural`
- `material.SetBuffer()`
- xorshift 随机算法
- 集群行为模拟
- `Graphics.DrawMeshInstancedIndirect`
- 旋转平移缩放矩阵，齐次坐标
- Surface Shader
- `ComputeBufferType.Default`
- `#pragma instancing_options procedural:setup`
- `unity_InstanceID`
- Skinned Mesh Renderer
- 数据对齐



### 1. 介绍与准备工作

Compute Shader除了可以同时处理大量的数据，还有一个关键的优势，就是Buffer存储在GPU中。因此可以将Compute Shader处理好的数据直接传递给与Material关联的Shader中，即Vertex/Fragment Shader。这里的关键就是，material也可以像Compute Shader一样`SetBuffer()`，直接从GPU的Buffer中访问数据！

![image-20240527145213065](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527145213065.png)

使用Compute Shader来制作粒子系统可以充分体现Compute Shader的强大并行能力。

在渲染过程中，Vertex Shader会从Compute Buffer中读取每个粒子的位置和其他属性，并将它们转换为屏幕上的顶点。Fragment Shader则负责根据这些顶点的信息（如位置和颜色）来生成像素。通过`Graphics.DrawProcedural`方法，Unity可以**直接渲染**这些由Shader处理的顶点，无需预先定义的网格结构，也不依赖Mesh Renderer，这对于渲染大量粒子特别有效。

### 2. 粒子你好

步骤也是非常简单，在 C# 中定义好粒子的信息（位置、速度与生命周期），初始化将数据传给Buffer，绑定Buffer到Compute Shader和Material。渲染阶段在`OnRenderObject()`里调用`Graphics.DrawProceduralNow`实现高效地渲染粒子。

![image-20240528170811317](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528170811317.png)

新建一个场景，制作一个效果：百万粒子跟随鼠标绽放生命的粒子，如下：

![image-20240527153558838](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527153558838.png)

写到这里，不禁让我思绪万千。粒子的生命周期很短暂，如同星火一般瞬间点燃，又如同流星一闪即逝。纵有千百磨难，我亦不过是亿万尘埃中的一粒，平凡且渺小。这些粒子，虽或许会在空间中随机漂浮（**使用"Xorshift"算法计算粒子生成的位置**），或许会拥有独一无二的色彩，但它们终究逃不出被程式预设的命运。这难道不正是我的人生写照吗？按部就班地上演着自己的角色，无法逃脱那无形的束缚。

> “上帝已死！而我们这些杀死他的人，又怎能不感到最大的痛苦呢？” - 弗里德里希·尼采

尼采不仅宣告了宗教信仰的消逝，更指出了现代人面临的虚无感，即没有了传统的道德和宗教支柱，人们感到了前所未有的孤独和方向感的缺失。粒子在C#脚本中被定义、创造，按照特定规则运动和消亡，这与尼采所描述的现代人在宇宙中的状态颇有相似之处。虽然每个人都试图寻找自己的意义，但最终仍受限于更广泛的社会和宇宙规则。

生活中充满了各种不可避免的痛苦，反映了人类存在的固有虚无和孤独感。失恋、生离死别、工作失意以及**即将编写的粒子死亡逻辑**等等，都印证了尼采所表达的，生活中没有什么是永恒不变的。同一个Buffer中的粒子必然在未来某个时刻消失，这体现了尼采所描述的现代人的孤独感，个体可能会感受到前所未有的孤立无援，因此每个人都是孤独的战士，必须学会独自面对内心的龙卷风和外部世界的冷漠。

但是没关系，「夏天会周而复始，该相逢的人会再次相逢」。本文的粒子也会在结束后再次生成，以最好的状态拥抱属于它的Buffer。

> Summer will come around again. People who meet will meet again.

![image-20240527153837421](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527153837421.png)

当前版本代码，可以自己拷下来跑跑（都有注释）：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_First_Particle/Assets/Shaders/ParticleFun.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_First_Particle/Assets/Scripts/ParticleFun.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_First_Particle/Assets/Shaders/Particle.shader

废话就说到这，先看看 C# 脚本是咋写的。

![image-20240527154208386](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527154208386.png)

老样子，先定义粒子的Buffer（结构体），并且初始化一下子，然后传给GPU，**关键在于最后三行将Buffer绑定给shader的操作**。下面省略号的代码没什么好讲的，都是常规操作，用注释一笔带过了。

```csharp
struct Particle{
    public Vector3 position; // 粒子位置
    public Vector3 velocity; // 粒子速度
    public float life;       // 粒子生命周期
}
ComputeBuffer particleBuffer; // GPU 的 Buffer
...
// Init() 中
    // 初始化粒子数组
    Particle[] particleArray = new Particle[particleCount];

    for (int i = 0; i < particleCount; i++){
        // 生成随机位置和归一化
		...
        // 设置粒子的初始位置和速度
		... 
        // 设置粒子的生命周期
        particleArray[i].life = Random.value * 5.0f + 1.0f;
    }
    // 创建并设置Compute Buffer
	...
    // 查找Compute Shader中的kernel ID
	...
    // 绑定Compute Buffer到shader
    shader.SetBuffer(kernelID, "particleBuffer", particleBuffer);
    material.SetBuffer("particleBuffer", particleBuffer);
    material.SetInt("_PointSize", pointSize);
```

关键的渲染阶段来了 `OnRenderObject()` 。`material.SetPass` 用于设置渲染材质通道。`DrawProceduralNow` 方法在不使用传统网格的情况下绘制几何体。`MeshTopology.Points` 指定了渲染的拓扑类型为点，GPU会把每个顶点作为一个点来处理，不会进行顶点之间的连线或面的形成。第二个参数 `1` 表示从第一个顶点开始绘制。`particleCount` 指定了要渲染的顶点数，这里是粒子的数量，即告诉GPU总共需要渲染多少个点。

```csharp
void OnRenderObject()
{
    material.SetPass(0);
    Graphics.DrawProceduralNow(MeshTopology.Points, 1, particleCount);
}
```

获取当前鼠标位置方法。OnGUI()这个方法每一帧可能调用多次。z值设为摄像机的近裁剪面加上一个偏移量，这里加14是为了得到一个更合适视觉深度的世界坐标（也可以自行调整）。

```csharp
void OnGUI()
{
    Vector3 p = new Vector3();
    Camera c = Camera.main;
    Event e = Event.current;
    Vector2 mousePos = new Vector2();

    // Get the mouse position from Event.
    // Note that the y position from Event is inverted.
    mousePos.x = e.mousePosition.x;
    mousePos.y = c.pixelHeight - e.mousePosition.y;

    p = c.ScreenToWorldPoint(new Vector3(mousePos.x, mousePos.y, c.nearClipPlane + 14));

    cursorPos.x = p.x;
    cursorPos.y = p.y;
}
```

上面已经将 `ComputeBuffer particleBuffer;` 传到了Compute Shader和Shader中。

先看看Compute Shader的数据结构。没什么特别的。

```glsl
// 定义粒子数据结构
struct Particle
{
	float3 position;  // 粒子的位置
	float3 velocity;  // 粒子的速度
	float life;       // 粒子的剩余生命时间
};

// 用于存储和更新粒子数据的结构化缓冲区，可从GPU读写
RWStructuredBuffer<Particle> particleBuffer;

// 从CPU设置的变量
float deltaTime;       // 从上一帧到当前帧的时间差
float2 mousePosition;  // 当前鼠标位置
```

![image-20240527171633636](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527171633636.png)

这里简单讲讲一个特别好用的随机数序列生成方法 xorshift 算法。一会将用来随机粒子的运动方向如上图，粒子会随机朝着三维的方向运动。

- 详细参考：https://en.wikipedia.org/wiki/Xorshift
- 原论文链接：https://www.jstatsoft.org/article/view/v008i14

这个算法03年由George Marsaglia提出，优点在于运算速度极快，并且非常节约空间。即使是最简单的Xorshift实现，其伪随机数周期也是相当长的。

基本操作是位移（shift）和异或（xor）。算法的名字也由此而来。它的核心是维护一个非零的状态变量，通过对这个状态变量进行一系列的位移和异或操作来生成随机数。

```glsl
// 用于生成随机数的状态变量
uint rng_state;

uint rand_xorshift() {
    // Xorshift algorithm from George Marsaglia's paper
    rng_state ^= (rng_state << 13);  // 将状态变量左移13位，然后与原状态进行异或
    rng_state ^= (rng_state >> 17);  // 将更新后的状态变量右移17位，再次进行异或
    rng_state ^= (rng_state << 5);   // 最后，将状态变量左移5位，进行最后一次异或
    return rng_state;                // 返回更新后的状态变量作为生成的随机数
}
```

**基本Xorshift** 算法的核心已在前面的解释中提到，不过不同的位移组合可以创建多种变体。原论文还提到了Xorshift128变体。使用128位的状态变量，通过四次不同的位移和异或操作更新状态。代码如下：

![image-20240527170946333](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527170946333.png)

```c
// c language Ver
uint32_t xorshift128(void) {
    static uint32_t x = 123456789;
    static uint32_t y = 362436069;
    static uint32_t z = 521288629;
    static uint32_t w = 88675123; 
    uint32_t t = x ^ (x << 11);
    x = y; y = z; z = w;
    w = w ^ (w >> 19) ^ (t ^ (t >> 8));
    return w;
}
```

可以产生更长的周期和更好的统计性能。这个变体的周期接近 $$2^{128}-1$$​​ ，非常厉害。

总的来说，这个算法用在游戏开发完全足够了，只是不适合用在密码学等领域。

在Compute Shader中使用这个算法时，需要注意Xorshift算法生成的随机数范围时uint32的的范围，需要再做一个映射( [0, 2^32-1] 映射到 [0, 1])：

```glsl
float tmp = (1.0 / 4294967296.0);  // 转换因子
rand_xorshift()) * tmp
```

而粒子运动方向是有符号的，因此只要在这个基础上减去0.5就好了。三个方向的随机运动：

```glsl
float f0 = float(rand_xorshift()) * tmp - 0.5;
float f1 = float(rand_xorshift()) * tmp - 0.5;
float f2 = float(rand_xorshift()) * tmp - 0.5;
float3 normalF3 = normalize(float3(f0, f1, f2)) * 0.8f; // 缩放了运动方向
```

每一个Kernel需要完成的内容如下：

- 先得到Buffer中上一帧的粒子信息
- 维护粒子Buffer（计算粒子速度，更新位置、生命值），写回Buffer
- 若生命值小于0，重新生成一个粒子

生成粒子，初始位置利用刚刚Xorshift得到的随机数，定义粒子的生命值，重置速度。

```glsl
// 设置粒子的新位置和生命值
particleBuffer[id].position = float3(normalF3.x + mousePosition.x, normalF3.y + mousePosition.y, normalF3.z + 3.0);
particleBuffer[id].life = 4;  // 重置生命值
particleBuffer[id].velocity = float3(0,0,0);  // 重置速度
```

最后是Shader的基本数据结构：

```glsl
struct Particle{
    float3 position;
    float3 velocity;
    float life;
};

struct v2f{
    float4 position : SV_POSITION;
    float4 color : COLOR;
    float life : LIFE;
    float size: PSIZE;
};
// particles' data
StructuredBuffer<Particle> particleBuffer;
```

然后在顶点着色器计算粒子的顶点色、顶点的Clip位置以及传输一个顶点大小的信息。

```glsl
v2f vert(uint vertex_id : SV_VertexID, uint instance_id : SV_InstanceID){
    v2f o = (v2f)0;

    // Color
    float life = particleBuffer[instance_id].life;
    float lerpVal = life * 0.25f;
    o.color = fixed4(1.0f - lerpVal+0.1, lerpVal+0.1, 1.0f, lerpVal);

    // Position
    o.position = UnityObjectToClipPos(float4(particleBuffer[instance_id].position, 1.0f));
    o.size = _PointSize;

    return o;
}
```

片元着色器计算插值颜色。

```glsl
float4 frag(v2f i) : COLOR{
    return i.color;
}
```

至此，就可以得到上面的效果。

![image-20240527180235573](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527180235573.png)

### 3. Quad粒子

上一节每一个粒子都只有一个点，没什么意思。现在把一个点变成一个Quad。在Unity中，没有Quad，只有两个三角形组成的假Quad。

开干，基于上面的代码。在 C# 中定义顶点，一个Quad的尺寸。

```csharp
// struct
struct Vertex
{
    public Vector3 position;
    public Vector2 uv;
    public float life;
}
const int SIZE_VERTEX = 6 * sizeof(float);
public float quadSize = 0.1f; // Quad的尺寸
```

![image-20240527185659448](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527185659448.png)

每一个粒子的的基础上，设置六个顶点的uv坐标，给顶点着色器用。并且按照Unity规定的顺序绘制。

```csharp
    index = i*6;
    //Triangle 1 - bottom-left, top-left, top-right
    vertexArray[index].uv.Set(0,0);
    vertexArray[index+1].uv.Set(0,1);
    vertexArray[index+2].uv.Set(1,1);
    //Triangle 2 - bottom-left, top-right, bottom-right
    vertexArray[index+3].uv.Set(0,0);
    vertexArray[index+4].uv.Set(1,1);
    vertexArray[index+5].uv.Set(1,0);
```

最后传递给Buffer。这里的 `halfSize` 目的是传给Compute Shader计算Quad的各个顶点位置用的。

```csharp
vertexBuffer = new ComputeBuffer(numVertices, SIZE_VERTEX);
vertexBuffer.SetData(vertexArray);
shader.SetBuffer(kernelID, "vertexBuffer", vertexBuffer);
shader.SetFloat("halfSize", quadSize*0.5f);

material.SetBuffer("vertexBuffer", vertexBuffer);
```

渲染阶段把点改为三角形，有六个点。

```csharp
void OnRenderObject()
{
    material.SetPass(0);
    Graphics.DrawProceduralNow(MeshTopology.Triangles, 6, numParticles);
}
```

在Shader中改一下设置，接收顶点数据。并且接收一张贴图用于显示。需要做alpha剔除。

```glsl
_MainTex("Texture", 2D) = "white" {}     
...
Tags{ "Queue"="Transparent" "RenderType"="Transparent" "IgnoreProjector"="True" }
LOD 200
Blend SrcAlpha OneMinusSrcAlpha
ZWrite Off
...
    struct Vertex{
        float3 position;
        float2 uv;
        float life;
    };
    StructuredBuffer<Vertex> vertexBuffer;
    sampler2D _MainTex;
    v2f vert(uint vertex_id : SV_VertexID, uint instance_id : SV_InstanceID)
    {
        v2f o = (v2f)0;

        int index = instance_id*6 + vertex_id;
        float lerpVal = vertexBuffer[index].life * 0.25f;
        o.color = fixed4(1.0f - lerpVal+0.1, lerpVal+0.1, 1.0f, lerpVal);
        o.position = UnityWorldToClipPos(float4(vertexBuffer[index].position, 1.0f));
        o.uv = vertexBuffer[index].uv;

        return o;
    }

    float4 frag(v2f i) : COLOR
    {
        fixed4 color = tex2D( _MainTex, i.uv ) * i.color;
        return color;
    }
```

在Compute Shader中，增加接收顶点数据，还有halfSize。

```glsl
struct Vertex
{
	float3 position;
	float2 uv;
	float life;
};
RWStructuredBuffer<Vertex> vertexBuffer;
float halfSize;
```

计算每个Quad六个顶点的位置。

![image-20240528115516507](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528115516507.png)

```glsl
	//Set the vertex buffer //
	int index = id.x * 6;
	//Triangle 1 - bottom-left, top-left, top-right   
	vertexBuffer[index].position.x = p.position.x-halfSize;
	vertexBuffer[index].position.y = p.position.y-halfSize;
	vertexBuffer[index].position.z = p.position.z;
	vertexBuffer[index].life = p.life;
	vertexBuffer[index+1].position.x = p.position.x-halfSize;
	vertexBuffer[index+1].position.y = p.position.y+halfSize;
	vertexBuffer[index+1].position.z = p.position.z;
	vertexBuffer[index+1].life = p.life;
	vertexBuffer[index+2].position.x = p.position.x+halfSize;
	vertexBuffer[index+2].position.y = p.position.y+halfSize;
	vertexBuffer[index+2].position.z = p.position.z;
	vertexBuffer[index+2].life = p.life;
	//Triangle 2 - bottom-left, top-right, bottom-right  // // 
	vertexBuffer[index+3].position.x = p.position.x-halfSize;
	vertexBuffer[index+3].position.y = p.position.y-halfSize;
	vertexBuffer[index+3].position.z = p.position.z;
	vertexBuffer[index+3].life = p.life;
	vertexBuffer[index+4].position.x = p.position.x+halfSize;
	vertexBuffer[index+4].position.y = p.position.y+halfSize;
	vertexBuffer[index+4].position.z = p.position.z;
	vertexBuffer[index+4].life = p.life;
	vertexBuffer[index+5].position.x = p.position.x+halfSize;
	vertexBuffer[index+5].position.y = p.position.y-halfSize;
	vertexBuffer[index+5].position.z = p.position.z;
	vertexBuffer[index+5].life = p.life;
```

大功告成。

![image-20240527190957824](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527190957824.png)

当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Quad/Assets/Shaders/QuadParticles.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Quad/Assets/Scripts/QuadParticles.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Quad/Assets/Shaders/QuadParticle.shader

下一节，将Mesh升级为预制体，并且尝试模拟鸟类飞行时的集群行为。

### 4. Flocking（集群行为）模拟

![image-20240528120157041](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528120157041.png)

Flocking 是一种模拟自然界中鸟群、鱼群等动物集体运动行为的算法。核心是基于三个基本的行为规则，由Craig Reynolds在Sig 87提出，通常被称为“Boids”算法：

- **分离（Separation）** 粒子与粒子之间不能太靠近，要有边界感。具体是计算周边一定半径的粒子然后计算一个避免碰撞的方向。
- **对齐（Alignment）** 个体的速度趋于群体的平均速度，要有归属感。具体是计算视觉范围内粒子的平均速度（速度大小 $$\cdot$$ **方向**）。这个视觉范围要根据鸟类实际的生物特性决定，下一节会提及。
- **聚合（Cohesion）** 个体的位置趋于平均位置（群体的中心），要有安全感。具体是，每个粒子找出周围邻居的几何中心，计算一个移动向量（最终结果是平均**位置**）。

![image-20240528112936752](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528112936752.png)

![image-20240528131504973](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528131504973.png)

思考一下，上面三个规则，哪一个最难实现？

答：Separation。众所周知，计算物体间的碰撞是非常难以实现的。因为每个个体都需要与其他所有个体进行距离比较，这会导致算法的时间复杂度接近O(n^2)，其中n是粒子的数量。例如，如果有1000个粒子，那么在每次迭代中可能需要进行将近500,000次的距离计算。在当年原论文作者在没有经过优化的原始算法（时间复杂度O(N^2)）中渲染一帧（80只鸟）所需时间是95秒，渲染一个300帧的动画使用了将近9个小时。

一般来说，使用四叉树或者是格点哈希（Spatial Hashing）等空间划分方法可以优化计算。也可以维护一个近邻列表存储每个个体周边一定距离的个体。当然了，还可以使用Compute Shader硬算。

![image-20240528114209032](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528114209032.png)

废话不多说，开干。

首先下载好预备的工程文件（如果没有事先准备）：

- 鸟的Prefab：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/main/Assets/Prefabs/Boid.prefab
- 脚本：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/main/Assets/Scripts/SimpleFlocking.cs
- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/main/Assets/Shaders/SimpleFlocking.compute

然后添加到一个空GO中。

![image-20240528114835489](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528114835489.png)

启动项目就可以看到一堆鸟。

![image-20240528114936634](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528114936634.png)

下面是关于群体行为模拟的一些参数。

```csharp
// 定义群体行为模拟的参数。
    public float rotationSpeed = 1f; // 旋转速度。
    public float boidSpeed = 1f; // Boid速度。
    public float neighbourDistance = 1f; // 邻近距离。
    public float boidSpeedVariation = 1f; // 速度变化。
    public GameObject boidPrefab; // Boid对象的预制体。
    public int boidsCount; // Boid的数量。
    public float spawnRadius; // Boid生成的半径。
    public Transform target; // 群体的移动目标。
```

除了Boid预制体`boidPrefab`和生成半径`spawnRadius`之外，其他都需要传给GPU。

为了方便，这一节先犯个蠢，只在GPU计算鸟的位置和方向，然后传回给CPU，做如下处理：

```csharp
...
boidsBuffer.GetData(boidsArray);
// 更新每个鸟的位置与朝向
for (int i = 0; i < boidsArray.Length; i++){
    boids[i].transform.localPosition = boidsArray[i].position;
    if (!boidsArray[i].direction.Equals(Vector3.zero)){
        boids[i].transform.rotation = Quaternion.LookRotation(boidsArray[i].direction);
    }
}
```

`Quaternion.LookRotation()` 方法用于创建一个旋转，使对象面向指定的方向。

在Compute Shader中计算每个鸟的位置。

```glsl
#pragma kernel CSMain
#define GROUP_SIZE 256    
struct Boid{
	float3 position;
	float3 direction;
};
RWStructuredBuffer<Boid> boidsBuffer;
float time;
float deltaTime;
float rotationSpeed;
float boidSpeed;
float boidSpeedVariation;
float3 flockPosition;
float neighbourDistance;
int boidsCount;
[numthreads(GROUP_SIZE,1,1)]
void CSMain (uint3 id : SV_DispatchThreadID){
	...// 接下文
}
```

先写对齐和聚合的逻辑，最终输出实际位置、方向给Buffer。

```glsl
	Boid boid = boidsBuffer[id.x];

	float3 separation = 0; // 分离
	float3 alignment = 0; // 对齐 - 方向
	float3 cohesion = flockPosition; // 聚合 - 位置

	uint nearbyCount = 1; // 自身算作周边的个体。

	for (int i=0; i<boidsCount; i++)
	{
		if(i!=(int)id.x) // 把自己排除 
		{
			Boid temp = boidsBuffer[i];
            // 计算周围范围内的个体
			if(distance(boid.position, temp.position)< neighbourDistance){
				alignment += temp.direction;
				cohesion += temp.position;
				nearbyCount++;
			}
		}
	}
	float avg = 1.0 / nearbyCount;
	alignment *= avg;
	cohesion *= avg;
	cohesion = normalize(cohesion-boid.position);

	// 综合一个移动方向
	float3 direction = alignment + separation + cohesion;
	// 平滑转向和位置更新
	boid.direction = lerp(direction, normalize(boid.direction), 0.94);
	// deltaTime确保移动速度不会因帧率变化而改变。
	boid.position += boid.direction * boidSpeed * deltaTime;

	boidsBuffer[id.x] = boid;
```

这就是没有边界感（分离项）的下场，所有的个体都表现出相当亲密的关系，都重叠在一起了。

![image-20240528131608696](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528131608696.png)

添加下面的代码。

```glsl
if(distance(boid.position, temp.position)< neighbourDistance)
{
    float3 offset = boid.position - temp.position;
    float dist = length(offset);
    if(dist < neighbourDistance)
    {
        dist = max(dist, 0.000001);
        separation += offset * (1.0/dist - 1.0/neighbourDistance);
    }
    ...
```

`1.0/dist` 当Boid越靠近时，这个值越大，表示分离力度应当越大。`1.0/neighbourDistance` 是一个常数，基于定义的邻近距离。两者的差值表示实际的分离力应对距离的反应程度。如果两个Boid的距离正好是 `neighbourDistance`，这个值为零（没有分离力）。如果两个Boid距离小于 `neighbourDistance`，这个值为正，且距离越小，值越大。

![image-20240528140142454](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528140142454.png)

当前代码：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Flocking/Assets/Shaders/SimpleFlocking.compute

下一节将采用Instanced Mesh，提高性能。

### 5. GPU Instancing优化

首先回顾一下本章节的内容。「粒子你好」与「Quad粒子」的两个例子中，我们都运用了Instanced技术（`Graphics.DrawProceduralNow()`），将Compute Shader的计算好的粒子位置直接传递给VertexFrag着色器。

![image-20240527145213065](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240527145213065.png)

本节使用的`DrawMeshInstancedIndirect` 用于绘制大量几何体实例，实例都是相似的，只是位置、旋转或其他参数略有不同。相对于每帧都重新生成几何体并渲染的 `DrawProceduralNow`，`DrawMeshInstancedIndirect` 只需要一次性设置好实例的信息，然后 GPU 就可以根据这些信息一次性渲染所有实例。渲染草地、群体动物就用这个函数。

![image-20240528170836520](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528170836520.png)

这个函数有很多参数，只用其中的一部分。

![image-20240528150757414](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528150757414.png)

```csharp
Graphics.DrawMeshInstancedIndirect(boidMesh, 0, boidMaterial, bounds, argsBuffer);
```

1. `boidMesh`：把鸟Mesh丢进去。
2. `subMeshIndex`：绘制的子网格索引。如果网格只有一个子网格，通常为0。
3. `boidMaterial`：应用到实例化对象的材质。
4. `bounds`：包围盒指定了绘制的范围。实例化对象只有在这个包围盒内的区域才会被渲染。优化性能之用。
5. `argsBuffer`：参数的 `ComputeBuffer`，参数包括每个实例的几何体的索引数量和实例化的数量。

这个 `argsBuffer` 是啥？这个参数用来告诉Unity，我们现在要渲染哪个Mesh、要渲染多少个！可以用一种特殊的Buffer作为参数给进去。

在初始化shader时候，创建一种特殊Buffer，其标注为 `ComputeBufferType.IndirectArguments` 。这种类型的缓冲区专门用于传递给 GPU，以便在 GPU 上执行间接绘制命令。这里的new ComputeBuffer 第一个参数是 `1` ，表示一个args数组（一个数组有5个uint），不要理解错了。

```csharp
ComputeBuffer argsBuffer;
...
argsBuffer = new ComputeBuffer(1, 5 * sizeof(uint), ComputeBufferType.IndirectArguments);
if (boidMesh != null)
{
    args[0] = (uint)boidMesh.GetIndexCount(0);
    args[1] = (uint)numOfBoids;
}
argsBuffer.SetData(args);
...
Graphics.DrawMeshInstancedIndirect(boidMesh, 0, boidMaterial, bounds, argsBuffer);
```

在上一章的基础上，个体的数据结构增加一个offset，在Compute Shader用于方向上的偏移。另外初始状态的方向用Slerp插值，70%保持原来的方向，30%随机。Slerp插值的结果是四元数，需要用四元数方法转换到欧拉角再传入构造函数。

```csharp
public float noise_offset;
...
Quaternion rot = Quaternion.Slerp(transform.rotation, Random.rotation, 0.3f);
boidsArray[i] = new Boid(pos, rot.eulerAngles, offset);
```

将这个新的属性`noise_offset`传到Compute Shader后，计算范围是 [-1, 1] 的噪声值，应用到鸟的速度上。

```glsl
float noise = clamp(noise1(time / 100.0 + boid.noise_offset), -1, 1) * 2.0 - 1.0;
float velocity = boidSpeed * (1.0 + noise * boidSpeedVariation);
```

然后稍微优化了一下算法。Compute Shader大体是没有区别的。

```glsl
if (distance(boid_pos, boidsBuffer[i].position) < neighbourDistance)
{
    float3 tempBoid_position = boidsBuffer[i].position;

    float3 offset = boid.position - tempBoid_position;
    float dist = length(offset);
    if (dist<neighbourDistance){
        dist = max(dist, 0.000001);//Avoid division by zero
        separation += offset * (1.0/dist - 1.0/neighbourDistance);
    }
    alignment += boidsBuffer[i].direction;
    cohesion += tempBoid_position;

    nearbyCount += 1;
}
```

最大的不同在于Shader上。本节使用Surface Shader取代Frag。这个东西其实就是一个包装好的vertex and fragment shader。Unity已经完成了光照、阴影等一系列繁琐的工作。你依旧可以指定一个Vert。

写Shader制作材质的时候，需要对Instanced的物体做特别处理。因为普通的渲染对象，他们的位置、旋转和其他属性在Unity中是静态的。而对于当前要构建的实例化对象，其位置、旋转等参数时刻在变化，因此，在渲染管线中需要通过特殊的机制来动态设置每个实例化对象的位置和参数。当前的方法基于程序的实例化技术，可以一次性渲染所有的实例化对象，而不需要逐个绘制。也就是一次性批量渲染。

着色器应用instanced技术方法。实例化阶段是在vert之前执行。这样每个实例化的对象都有单独的旋转、位移和缩放等矩阵。

现在需要为每个实例化对象创建属于他们的旋转矩阵。从Buffer中我们拿到了Compute Shader计算后的鸟的基本信息（上一节中，该数据传回了CPU，这里直接传给Shader做实例化）：

![image-20240528180501947](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528180501947.png)

Shader里将Buffer传来的数据结构、相关操作用下面的宏包裹起来。

```glsl
// .shader
#ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
struct Boid
{
    float3 position;
    float3 direction;
    float noise_offset;
};

StructuredBuffer<Boid> boidsBuffer; 
#endif
```

由于我只在 C# 的 `DrawMeshInstancedIndirect` 的`args[1]`指定了需要实例化的数量（鸟的数量，也是Buffer的大小），因此直接使用`unity_InstanceID`索引访问Buffer就好了。

```glsl
#pragma instancing_options procedural:setup
    
void setup()
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
        _BoidPosition = boidsBuffer[unity_InstanceID].position;
        _Matrix = create_matrix(boidsBuffer[unity_InstanceID].position, boidsBuffer[unity_InstanceID].direction, float3(0.0, 1.0, 0.0));
    #endif
}
```

这里的空间变换矩阵的计算涉及到**Homogeneous Coordinates**，可以去复习一下GAMES101的课程。点是(x,y,z,1)，坐标是(x,y,z,0)。

如果使用仿射变换（Affine Transformations），代码是这样的：

```glsl
void setup()
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    _BoidPosition = boidsBuffer[unity_InstanceID].position;
    _LookAtMatrix = look_at_matrix(boidsBuffer[unity_InstanceID].direction, float3(0.0, 1.0, 0.0));
    #endif
}
 void vert(inout appdata_full v, out Input data)
{
    UNITY_INITIALIZE_OUTPUT(Input, data);
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    v.vertex = mul(_LookAtMatrix, v.vertex);
    v.vertex.xyz += _BoidPosition;
    #endif
}
```

不够优雅，我们直接使用一个齐次坐标（Homogeneous Coordinates）。一个矩阵搞掂旋转平移缩放！

```glsl
void setup()
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    _BoidPosition = boidsBuffer[unity_InstanceID].position;
    _Matrix = create_matrix(boidsBuffer[unity_InstanceID].position, boidsBuffer[unity_InstanceID].direction, float3(0.0, 1.0, 0.0));
    #endif
}
 void vert(inout appdata_full v, out Input data)
{
    UNITY_INITIALIZE_OUTPUT(Input, data);

    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
    v.vertex = mul(_Matrix, v.vertex);
    #endif
}
```

至此，就大功告成了！当前的帧率比上一节提升了将近一倍。

![image-20240528155421917](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528155421917.png)

![image-20240528155819234](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528155819234.png)

当前版本代码：

- Compute Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Instanced/Assets/Shaders/InstancedFlocking.compute
- CPU：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Instanced/Assets/Scripts/InstancedFlocking.cs
- Shader：https://github.com/Remyuu/Unity-Compute-Shader-Learn/blob/L4_Instanced/Assets/Shaders/InstancedFlocking.shader

### 6. 应用蒙皮动画

![image-20240528162512809](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528162512809.png)

本节要做的是，使用Animator组件，在实例化物体之前，将各个关键帧的Mesh抓取到Buffer当中。通过选取不同索引，得到不同姿势的Mesh。具体的骨骼动画制作不在本文讨论范围。

只需要在上一章的基础上修改代码，添加Animator等逻辑。我已经在下面写了注释，可以看看。

并且个体的数据结构有所更新：

```csharp
struct Boid{
    float3 position;
    float3 direction;
    float noise_offset;
    float speed; // 暂时没啥用
    float frame; // 表示动画中的当前帧索引
    float3 padding; // 确保数据对齐
};
```

详细说说这里的对齐。一个数据结构中，数据的大小最好是16字节的整数倍。

- `float3 position;` (12字节)
- `float3 direction;` (12字节)
- `float noise_offset;` (4字节)
- `float speed;` (4字节)
- `float frame;` (4字节)
- `float3 padding;` (12字节)

如果没有Padding，大小是36字节，不是常见的对齐大小。加上Padding，对齐到48字节，完美！

```csharp
private SkinnedMeshRenderer boidSMR; // 用于引用包含蒙皮网格的SkinnedMeshRenderer组件。
private Animator animator;
public AnimationClip animationClip; // 具体的动画剪辑，通常用于计算动画相关的参数。

private int numOfFrames; // 动画中的帧数，用于确定在GPU缓冲区中存储多少帧数据。
public float boidFrameSpeed = 10f; // 控制动画播放的速度。
MaterialPropertyBlock props; // 在不创建新材料实例的情况下传递参数给着色器。这意味着可以改变实例的材质属性（如颜色、光照系数等），而不会影响到使用相同材料的其他对象。
Mesh boidMesh; // 存储从SkinnedMeshRenderer烘焙出的网格数据。
...
void Start(){ // 这里首先初始化Boid数据，然后调用GenerateSkinnedAnimationForGPUBuffer来准备动画数据，最后调用InitShader来设置渲染所需的Shader参数。
    ...
    // This property block is used only for avoiding an instancing bug.
    props = new MaterialPropertyBlock();
    props.SetFloat("_UniqueID", Random.value);
	...
    InitBoids();
    GenerateSkinnedAnimationForGPUBuffer();
    InitShader();
}
void InitShader(){ // 此方法配置Shader和材料属性，确保动画播放可以根据实例的不同阶段正确显示。frameInterpolation的启用或禁用决定了是否在动画帧之间进行插值，以获得更平滑的动画效果。
    ...
    if (boidMesh)//Set by the GenerateSkinnedAnimationForGPUBuffer
    ...
    shader.SetFloat("boidFrameSpeed", boidFrameSpeed);
    shader.SetInt("numOfFrames", numOfFrames);
    boidMaterial.SetInt("numOfFrames", numOfFrames);
    if (frameInterpolation && !boidMaterial.IsKeywordEnabled("FRAME_INTERPOLATION"))
    boidMaterial.EnableKeyword("FRAME_INTERPOLATION");
	if (!frameInterpolation && boidMaterial.IsKeywordEnabled("FRAME_INTERPOLATION"))
    boidMaterial.DisableKeyword("FRAME_INTERPOLATION");
}
void Update(){
	...
    // 后面两个参数：
        // 1. 0: 参数缓冲区的偏移量，用于指定从哪里开始读取参数。
        // 2. props: 前面创建的 MaterialPropertyBlock，包含所有实例共享的属性。
    Graphics.DrawMeshInstancedIndirect( boidMesh, 0, boidMaterial, bounds, argsBuffer, 0, props);
}
void OnDestroy(){ 
    ...
    if (vertexAnimationBuffer != null) vertexAnimationBuffer.Release();
}
private void GenerateSkinnedAnimationForGPUBuffer()
{
	... // 接下文
}
```

为了给Shader在不同的时间提供不同姿势的Mesh，因此在 `GenerateSkinnedAnimationForGPUBuffer()` 函数中，从 `Animator` 和 `SkinnedMeshRenderer` 中提取每一帧的网格顶点数据，然后将这些数据存储到GPU的 `ComputeBuffer` 中，以便在实例化渲染时使用。

 通过`GetCurrentAnimatorStateInfo`获取当前动画层的状态信息，用于后续控制动画的精确播放。

`numOfFrames` 使用最接近动画长度和帧率乘积的二次幂来确定，可以优化GPU的内存访问。

然后创建一个`ComputeBuffer`来存储所有帧的所有顶点数据。`vertexAnimationBuffer`

在for循环中，烘焙所有动画帧。具体做法是，在每个`sampleTime`时间点播放并立即更新，然后烘焙当前动画帧的网格到`bakedMesh`中。并且提取刚刚烘焙好的Mesh顶点，更新到数组 `vertexAnimationData` 中，最后上传至GPU，结束。

```csharp
// ...接上文
boidSMR = boidObject.GetComponentInChildren<SkinnedMeshRenderer>();
boidMesh = boidSMR.sharedMesh;
animator = boidObject.GetComponentInChildren<Animator>();
int iLayer = 0;
AnimatorStateInfo aniStateInfo = animator.GetCurrentAnimatorStateInfo(iLayer);

Mesh bakedMesh = new Mesh();
float sampleTime = 0;
float perFrameTime = 0;

numOfFrames = Mathf.ClosestPowerOfTwo((int)(animationClip.frameRate * animationClip.length));
perFrameTime = animationClip.length / numOfFrames;

var vertexCount = boidSMR.sharedMesh.vertexCount;
vertexAnimationBuffer = new ComputeBuffer(vertexCount * numOfFrames, 16);
Vector4[] vertexAnimationData = new Vector4[vertexCount * numOfFrames];
for (int i = 0; i < numOfFrames; i++)
{
    animator.Play(aniStateInfo.shortNameHash, iLayer, sampleTime);
    animator.Update(0f);

    boidSMR.BakeMesh(bakedMesh);

    for(int j = 0; j < vertexCount; j++)
    {
        Vector4 vertex = bakedMesh.vertices[j];
        vertex.w = 1;
        vertexAnimationData[(j * numOfFrames) +  i] = vertex;
    }

    sampleTime += perFrameTime;
}

vertexAnimationBuffer.SetData(vertexAnimationData);
boidMaterial.SetBuffer("vertexAnimation", vertexAnimationBuffer);

boidObject.SetActive(false);
```

在Compute Shader中，维护每一个个体数据结构中储存的帧变量。

```glsl
boid.frame = boid.frame + velocity * deltaTime * boidFrameSpeed;
if (boid.frame >= numOfFrames) boid.frame -= numOfFrames;
```

在Shader中lerp不同帧的动画。

```glsl
void vert(inout appdata_custom v)
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
        #ifdef FRAME_INTERPOLATION
            v.vertex = lerp(vertexAnimation[v.id * numOfFrames + _CurrentFrame], vertexAnimation[v.id * numOfFrames + _NextFrame], _FrameInterpolation);
        #else
            v.vertex = vertexAnimation[v.id * numOfFrames + _CurrentFrame];
        #endif
        v.vertex = mul(_Matrix, v.vertex);
    #endif
}
void setup()
{
    #ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
        _Matrix = create_matrix(boidsBuffer[unity_InstanceID].position, boidsBuffer[unity_InstanceID].direction, float3(0.0, 1.0, 0.0));
        _CurrentFrame = boidsBuffer[unity_InstanceID].frame;
        #ifdef FRAME_INTERPOLATION
            _NextFrame = _CurrentFrame + 1;
            if (_NextFrame >= numOfFrames) _NextFrame = 0;
            _FrameInterpolation = frac(boidsBuffer[unity_InstanceID].frame);
        #endif
    #endif
}
```

<video src="https://vdn.vzuu.com/HD/bd9a64d2-1f01-11ef-9ebb-d2903382a75a-v8_f2_t1_M5aXDNP7.mp4?disable_local_cache=1&bu=1513c7c2&c=avc.8.0&f=mp4&expiration=1717131433&auth_key=1717131433-0-0-b3fb49f553c50358d8831dff9c01254d&v=ali&pu=e59e796c&pp=ChMxNDAxNjIzODY1NzM5NTc5MzkyGGMiC2ZlZWRfY2hvaWNlMhMxMzY5MDA1NjA4NTk5OTA0MjU3PXu830Q%3D&pf=Web&pt=zhihu"></video>

非常不容易，终于完整了。

![image-20240528205547355](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528205547355.png)

完整工程链接：https://github.com/Remyuu/Unity-Compute-Shader-Learn/tree/L4_Skinned/Assets/Scripts

### 8. 总结/小测试



When rendering points which gives the best answer?

![image-20240528190257535](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528190257535.png)



What are the three key steps in flocking?

![image-20240528190337302](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528190337302.png)



When creating an arguments buffer for DrawMeshInstancedIndirect, how many uints are required?

![image-20240528190350879](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528190350879.png)



We created the wing flapping by using a skinned mesh shader. True or False.

![image-20240528190509504](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528190509504.png)



In a shader used by DrawMeshInstancedIndirect, which variable name gives the correct index for the instance?

![image-20240528190447640](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20240528190447640.png)

## References

1. https://en.wikipedia.org/wiki/Boids

2. [Flocks, Herds, and Schools: A Distributed Behavioral Model](https://dl.acm.org/doi/10.1145/37401.37406)
