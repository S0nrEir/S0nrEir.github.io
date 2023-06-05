---
title: 【译】Unity博客《Accessing texture data efficiently》
tags: ["翻译","纹理","API"]
---

原文链接：[Accessing texture data efficiently](https://blog.unity.com/engine-platform/accessing-texture-data-efficiently)

本篇文章介绍了Unity中一些访问纹理数据的方法和API，以及他们之间在不同情况下的优劣和区别。

### 在Unity中使用纹理数据

像素数据描述了纹理中单个像素的颜色。Unity 提供了一些方法，使您能够使用 C# 脚本从像素数据中读取或写入。

您可以使用这些方法来复制或更新纹理（例如，向玩家的个人资料图片添加细节），或以特定方式使用纹理的数据，例如读取表示世界地图的纹理以确定在哪里放置对象。

有几种编写从像素数据中读取或写入代码的方法。您选择的方法取决于您计划对数据进行什么操作以及项目的性能需求。

本博客和附带的示例项目旨在帮助您浏览可用的 API 和常见性能陷阱。对两者的理解将帮助您编写高效的解决方案或在出现性能瓶颈时解决问题。

#### CPU和GPU的像素数据副本

对于大多数类型的纹理，Unity 存储两份像素数据副本：一份存储在 GPU 内存中，这是渲染所必需的，另一份存储在 CPU 内存中。这个副本是可选的，允许您从 CPU 中读取、写入和操作像素数据。存储有其像素数据副本的纹理称为可读纹理。需要注意的一点是，RenderTexture 仅存在于 GPU 内存中。

------

### CPU和GPU之间的区别

#### 内存

大多数硬件上，CPU 可用的内存与 GPU 不同。一些设备具有某种形式的部分共享内存（比如手机），但对于本博客，我们将假设经典的 PC 配置，其中 CPU 仅直接访问插入主板的 RAM，而 GPU 则依赖于自己的视频 RAM（VRAM）。在这些不同的环境之间传输的任何数据都必须通过 PCI 总线传输，这比在相同类型的内存中传输数据要慢。由于这些成本，您应该尝试限制每帧传输的数据量。

![](https://blog-api.unity.com/sites/default/files/2023-05/Texture%20API%20blogpost-API.drawio.png?imwidth=768&)

这张图描述了CPU和GPU内存之间的关系，以及他们如何通过API进行交互

#### 处理（纹理数据）

在shader中对纹理进行采样是最常见的 GPU 像素数据操作。要更改此数据，您可以在纹理之间复制或使用着色器渲染到纹理中。所有这些操作都可以由 GPU 快速执行。

在某些情况下，最好在 CPU 上操作纹理数据，因为这样可以更灵活地访问数据。CPU 像素数据操作仅对数据的 CPU 副本起作用，因此需要可读取的纹理。如果您想在着色器中采样更新后的像素数据，则必须先通过调用 Apply 将其从 CPU 复制到 GPU。根据涉及的纹理和操作的复杂性，坚持使用 CPU 操作可能更快、更容易（例如，在将几个 2D 纹理复制到 Texture2DArray Asset时）。

Unity API 提供了几种访问或处理纹理数据的方法。如果两者都存在，则某些操作会对 GPU 和 CPU 副本同时起作用。因此，这些方法的性能取决于纹理是否可读取。不同的方法可以用来实现相同的结果，但每种方法都有其自己的性能和易用性特征。

以下问题可以帮助你确定如何使用纹理API：

- GPU 是否能比 CPU 更快地执行您的计算？

  - 进程对纹理缓存施加了多大的压力？（例如，采样许多高分辨率纹理而不使用 mipmaps 可能会减慢 GPU 的速度。）

  - 进程是否需要随机写入纹理，或者可以输出到颜色或深度附件？（在纹理上随机写入像素需要频繁的缓存刷新，这会减慢进程。）

- 我的项目是否已经受到 GPU 瓶颈的影响？即使 GPU 能够比 CPU 更快地执行进程，GPU 能否承担更多的工作而不超过其帧时间预算？
  - 如果 GPU 和 CPU 主线程都接近其帧时间限制，则进程的缓慢部分可能可以由 CPU 工作线程执行。

- 需要上传到 GPU 或从 GPU 下载多少数据以计算或处理结果？
  - 着色器或 C# job pack(？)是否可以将数据打包成较小的格式以减少所需的带宽？
    - RenderTexture 是否可以降采样为下载的较小分辨率版本？

- 进程是否可以分块执行？（如果需要一次处理大量数据，则存在 GPU 没有足够内存来处理它的风险。）
  - 需要多快获得结果？计算或数据传输是否可以异步执行并稍后处理？（如果在单个帧中完成了太多工作，则存在 GPU 没有足够时间为每个帧渲染实际图形的风险。）

#### 将纹理设置为可读或不可读

默认情况下，导入到项目中的纹理资源是不可读取的，而从脚本创建的纹理是可读取的。

可读取的纹理使用的内存是不可读取的纹理的两倍，因为它们需要在 CPU RAM 中拥有其像素数据的副本。只有在需要时才应该使纹理可读取，并在处理完 CPU 上的数据后使它们不可读取。

要查看项目中的纹理资源是否可读取并进行编辑，请使用 Texture Import Settings 中的 Read/Write Enabled 选项或 TextureImporter.isReadable API。

要使纹理不可读取，请调用其 Apply 方法，并将 makeNoLongerReadable 参数设置为“true”（例如，Texture2D.Apply 或 Cubemap.Apply）。不可读取的纹理无法再次变为可读取。

在编辑和播放模式下，所有纹理对编辑器都是可读取的。调用 Apply 使纹理不可读取将更新 isReadable 的值，防止您访问 CPU 数据。但是，一些 Unity 进程将像纹理是可读取一样运行，因为它们看到内部 CPU 数据是有效的。

#### GITHUB上的官方例子

在访问纹理数据的各种方式之间，特别是在 CPU 上（尽管在较低分辨率下差异较小），性能差异很大。GitHub 上的 Unity Texture Access API 示例存储库包含了许多示例，展示了允许访问或操作纹理数据的各种 API 之间的性能差异。UI 仅显示主线程 CPU 计时。在某些情况下，DOTS 功能（如 Burst 和作业系统）被用于最大化性能。

以下是 GitHub 存储库中包含的示例：[UnityTextureAccessApiExamples](https://github.com/Unity-Technologies/UnityTextureAccessApiExamples/tree/2023-best-practices-article)

- SimpleCopy：从一个纹理复制所有像素到另一个纹理 
- PlasmaTexture：每帧在 CPU 上更新的Plasma纹理 
- TransferGPUTexture：将 GPU 上一个纹理中的所有像素（复制到不同大小或格式）传输到 RenderTexture 

以下是从 GitHub 示例中获取的性能测量。

测试设备为 3.7 GHz 8 核 Xeon® W-2145 CPU 和 RTX 2080。

#### 复制

这些是 SimpleCopy.UpdateTestCase 的中位 CPU 时间，纹理大小为 2,048。

请注意，由于 Graphics 方法只是将工作推送到 RenderThread 上，因此它们几乎立即在主线程上完成，并在下一帧渲染时准备好结果。

结果 ：

- 1,326 ms- foreach(mip) for(x in width) for(y in height) SetPixel(x, y, GetPixel(x, y, mip), mip) 
- 32.14 ms - foreach(mip) SetPixels(source.GetPixels(mip), mip) 
- 6.96 ms - foreach(mip) SetPixels32(source.GetPixels32(mip), mip) 
- 6.74 ms - LoadRawTextureData(source.GetRawTextureData()) 
- 3.54 ms - Graphics.CopyTexture(readableSource, readableTarget) 
- 2.87 ms - foreach(mip) SetPixelData<byte>(mip, GetPixelData<byte>(mip)) \
- 2.87 ms - LoadRawTextureData(source.GetRawTextureData<byte>()) 
- 0.00 ms - Graphics.ConvertTexture(source, target) 
- 0.00 ms - Graphics.CopyTexture(nonReadableSource, target)

#### Plasma纹理

这些是纹理大小为 512 时 PlasmaTexture.UpdateTestCase 的中位 CPU 时间。

你会注意到，SetPixels32 比 SetPixels 慢得出乎意料。这是由于必须将等离子体像素计算的基于浮点数的 Color 结果转换为基于字节的 Color32 结构。SetPixels32NoConversion 跳过此转换，只为 Color32 输出数组分配默认值，从而比 SetPixels 更具性能。为了打败 SetPixels 和 Unity 执行的底层颜色转换的性能，有必要重新设计像素计算方法本身，以直接输出 Color32 值。使用 SetPixelData 的简单实现几乎可以保证比仔细的 SetPixels 和 SetPixels32 方法给出更好的结果。

结果：

- 126.95 ms - SetPixel 
- 113.16 ms - SetPixels32 
- 88.96 ms - SetPixels 
- 86.30 ms - SetPixels32NoConversion 
- 16.91 ms - SetPixelDataBurst 
- 4.27 ms - SetPixelDataBurstParallel

#### TransferGPUTexture

这些是纹理大小为 8,196 时 TransferGPUTexture.UpdateTestCase 的 Editor GPU 时间：

- Blit - 1.584 ms 
- CopyTexture - 0.882 ms 



### 像素数据相关 API 建议 

您可以以各种方式访问像素数据。但是，并非所有方法都支持每种格式、纹理类型或用例，有些方法的执行时间比其他方法长。本节介绍了推荐的方法，下一节介绍了需要谨慎使用的方法。

#### CopyTexture

CopyTexture 是将 GPU 数据从一个纹理传输到另一个纹理的最快方法。它不执行任何格式转换。您可以通过指定源和目标位置以及区域的宽度和高度来部分复制数据。如果两个纹理都可读，则复制操作也将在 CPU 数据上执行，将此方法的总成本与仅使用 SetPixelData 和源纹理的 GetPixelData 结果进行 CPU-only 复制的成本更接近。

#### Blit

Blit 是一种快速而强大的方法，使用着色器将 GPU 数据传输到 RenderTexture 中。在实践中，这需要设置图形管道 API 状态以渲染到目标 RenderTexture。与 CopyTexture 相比，它具有较小的分辨率无关设置成本。该方法使用的默认 Blit 着色器接受输入纹理并将其渲染到目标 RenderTexture 中。通过提供自定义材质或着色器，您可以定义复杂的纹理到纹理渲染过程。

#### GetPixelData 和 SetPixelData 

GetPixelData 和 SetPixelData（以及 GetRawTextureData）是仅涉及 CPU 数据时使用的最快方法。这两种方法都要求您提供一个结构类型作为模板参数来重新解释数据。方法本身只需要此结构来推导出正确的大小，因此如果您不想定义表示纹理格式的自定义结构，则可以只使用 byte。

在访问单个像素时，最好定义一个自定义结构，并提供一些实用程序方法以方便使用。例如，可以由一个 ushort 数据成员组成 R5G5B5A1 格式结构，并提供一些 get/set 方法以访问单个通道作为字节。

````c#
public struct FormatR5G5B5A1
{
        public ushort data;

        const ushort redOffset = 11;
        const ushort greenOffset = 6;
        const ushort blueOffset = 1;
        const ushort alphaOffset = 0;

        const ushort redMask = 31 << redOffset;
        const ushort greenMask = 31 << greenOffset;
        const ushort blueMask = 31 << blueOffset;
        const ushort alphaMask = 1;

        public byte red { get { return (byte)((data & redMask) >> redOffset); } }
        public byte green { get { return (byte)((data & greenMask) >> greenOffset); } }
        public byte blue { get { return (byte)((data & blueMask) >> blueOffset); } }
        public byte alpha { get { return (byte)((data & alphaMask) >> alphaOffset); } }
}
````

这段代码是 R5G5B5A5A1 格式像素对象的实现示例；为了简洁起见，相应的属性设置器被省略。

SetPixelData 可用于将完整的 Mip 级别数据复制到目标纹理中。GetPixelData 将返回一个 NativeArray，该数组实际上指向 Unity 内部 CPU 纹理数据的一个 Mip 级别。这使您可以直接读取/写入该数据，而无需进行任何复制操作。但是，GetPixelData 返回的 NativeArray 仅保证在用户代码调用 GetPixelData 并返回控制权给 Unity 时（例如当 MonoBehaviour.Update 返回时）有效。您不能在帧之间存储 GetPixelData 的结果，而是必须为要从中访问此数据的每个帧获取正确的 NativeArray。

#### Apply 

Apply 方法在 CPU 数据上传到 GPU 后返回。makeNoLongerReadable 参数应在可能的情况下设置为“true”，以在上传后释放 CPU 数据的内存。

#### RequestIntoNativeArray 和 RequestIntoNativeSlice

RequestIntoNativeArray 和 RequestIntoNativeSlice 方法异步地将 GPU 数据从指定的纹理下载到用户提供的 NativeArray（片段）中。

调用这些方法将返回一个请求句柄，该句柄可以指示所请求的数据是否已下载完成。支持的格式仅有少数几种，因此请使用 SystemInfo.IsFormatSupported 和 FormatUsage.ReadPixels 检查格式支持。AsyncGPUReadback 类还具有 Request 方法，该方法为您分配一个 NativeArray。如果需要重复此操作，则如果分配并重用 NativeArray，则可以获得更好的性能。

------

### 需要谨慎使用的方法

由于可能会产生显著的性能影响，因此有许多方法应谨慎使用。让我们更详细地看一下它们。

#### 具有底层转换的像素访问器 

这些方法执行各种复杂度的像素格式转换。Pixels32 变体是其中最高效的，但即使它们的底层纹理格式与 Color32 结构完全匹配，它们仍然可以执行格式转换。在使用以下方法时，最好记住随着像素数量的增加，它们的性能影响会显著增加：

- GetPixel
- GetPixelBilinear 
- SetPixel 
- GetPixels 
- SetPixels 
- GetPixels32 SetPixels32

#### 带有Catch的快速数据访问器 （？）

GetRawTextureData 和 LoadRawTextureData 是仅适用于 Texture2D 的方法，它们使用包含所有 Mip 级别原始像素数据的数组，一个接一个地进行处理。布局从最大到最小的 Mip，每个 Mip 都是“宽度”像素值的“高度”量。这些函数可以快速提供 CPU 数据访问。GetRawTextureData 有一个“陷阱”（？），即非模板化变体返回数据的副本。这会稍微慢一些，并且不允许直接操作由 Unity 管理的底层缓冲区。GetPixelData 没有这个问题，并且只能返回指向底层缓冲区的 NativeArray，该缓冲区在用户代码返回控制权给 Unity 之前保持有效。

#### ConvertTexture

ConvertTexture 是一种将 GPU 数据从一个纹理传输到另一个纹理的方法，其中源纹理和目标纹理的大小或格式不相同。在这种情况下，转换过程尽可能高效，但并不便宜。这是内部过程：

- 分配一个与目标纹理匹配的临时 RenderTexture
- 从源纹理到临时 RenderTexture 执行 Blit
- 将 Blit 结果从临时 RenderTexture 复制到目标纹理

以下问题可以帮助你确定是否需要使用该方法：

- 我需要执行此转换吗？
-  我能确保源纹理在导入时以所需的大小/格式为目标平台创建吗？
-  我能更改我的流程以使用相同的格式，允许一个流程的结果直接用作另一个流程的输入吗？ 
- 我能创建并使用 RenderTexture 作为目标吗？这样做将把转换过程减少到单个 Blit 到目标 RenderTexture。

#### ReadPixels

ReadPixels 方法会将 GPU 数据同步下载到活动 RenderTexture（RenderTexture.active）中，然后将其下载到 Texture2D 的 CPU 数据中。这使您可以存储或处理渲染操作的输出。支持的格式有限，因此请使用 SystemInfo.IsFormatSupported 和 FormatUsage.ReadPixels 检查格式支持。

从 GPU 下载数据是一个缓慢的过程。在它开始之前，ReadPixels 必须等待 GPU 完成所有前面的工作。最好避免使用此方法，因为它不会返回，直到请求的数据可用，这会降低性能。可用性也是一个问题，因为您需要 GPU 数据在 RenderTexture 中，并且必须将其配置为当前活动状态。当使用前面讨论过的 AsyncGPUReadback 方法时，可用性和性能都更好。

#### 转换图片文件格式

ImageConversion 类具有将 Texture2D 和多种图像文件格式之间进行转换的方法。LoadImage 能够将 JPG、PNG 或 EXR（自 2023.1 起）数据加载到 Texture2D 中，并为您上传到 GPU。根据 Texture2D 的原始格式，加载的像素数据可以在运行时压缩。其他方法可以将 Texture2D 或像素数据数组转换为 JPG、PNG、TGA 或 EXR 数据。

这些方法并不特别快，但如果您的项目需要通过常见图像文件格式传递像素数据，则可能很有用。典型用例包括从磁盘加载用户头像并通过网络与其他玩家共享。

本片文章的一些关键摘要：

以下是需要记住的关键点摘要：

- 在操作纹理时，第一步是评估哪些操作可以在 GPU 上执行以获得最佳性能。现有的 CPU/GPU 工作负载和输入/输出数据的大小是需要考虑的关键因素。 

- 使用低级函数（例如 GetRawTextureData）（意思应该是更底层的函数）在必要时实现特定的转换路径，可以比执行（通常是冗余的）复制和转换的更方便的方法提供更好的性能。 
- 更复杂的操作（例如大型读取和像素计算）只有在异步或并行执行时才适用于 CPU。Burst 和Job System的组合允许 C# 执行某些操作，否则这些操作只能在 GPU 上执行。 
- 频繁进行分析：在开发过程中，您可能会遇到许多陷阱，从意外和不必要的转换到等待另一个进程而导致停顿。某些性能问题只有在游戏扩展并且代码的某些部分使用更重时才会开始浮出水面。示例项目演示了看似小小的纹理分辨率增加如何导致某些 API 成为性能问题。