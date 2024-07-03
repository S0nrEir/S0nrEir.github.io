---
title: 译：关于Unity增量构建和AssetBundle哈希
tags: ["AssetBundle"]

---

本篇blog介绍了Unity在做增量打包时哈希的对比依据，以及这样做的风险

[原文地址在此](https://forum.unity.com/threads/about-incremental-build-and-asset-bundle-hashes.1436032/)

> The BuildPipeline.BuildAssetBundles() API is widely used in currently supported versions of Unity to build AssetBundles. This post describes how the AssetBundle Hash and Incremental Build works in the context of that API, and gives some recommendations based on some known limitations.
>
> In particular we recommend:
>
> - Doing clean builds when building official releases.
> - When calling UnityWebRequestAssetBundle(), avoid using the AssetBundle hash as the version.
> - (Updated) Starting with 2022.3.8f1 you can use BuildAssetBundleOptions.UseContentHash flag when building bundles, which makes the AssetBundle hash safer for use with UnityWebRequestAssetBundle().
>
> The rest of this post explains why these recommendations arise, by diving into some details of how the build, hashing and caching work.

BuildPipeline.BuildAssetBundles() API在目前支持的Unity版本中被广泛使用以构建AssetBundles。这篇文章描述了在该API的背景下，AssetBundle哈希和增量构建是如何工作的，并根据一些已知的限制提供了一些建议。

特别是我们建议：

- 当构建官方发布版时，进行全新构建（不使用增量）。
- 调用UnityWebRequestAssetBundle()时，避免使用AssetBundle哈希作为版本。
- (已更新）从2022.3.8f1开始，您可以在构建包时使用BuildAssetBundleOptions.UseContentHash标记，这使得AssetBundle哈希在与UnityWebRequestAssetBundle()一起使用时更安全。

这篇文章的剩余部分将通过深入了解构建、哈希和缓存的工作方式来解释为什么会有这些建议。

---

> **How does Incremental Build work?**
> In the BuildPipeline.BuildAssetBundles() implementation, the AssetBundle hash is used to capture the contents and dependencies of the AssetBundle.The flow is as follows:
>
> - Calculate the AssetBundle hash based on the current inputs (see later section for details)
> - If there is a .manifest file from a previous build of the AssetBundle then load the hash and compare
> - If the hashes match then calculate the TypeTreeHash
> - If the AssetBundle Hash and TypeTreeHash both match then do not rebuild the AssetBundle (unless the ForceRebuildAssetBundle flag is specified)
> - If the AssetBundle is built then the new AssetBundle and TypeTree hashes are serialized in the newly generated .manifest file.
>
> This is an example Manifest, showing the data that supports the incremental build.
>
> ```yaml
>  
> ManifestFileVersion: 0
> CRC: 2088487739
> Hashes:
>   AssetFileHash:
>     serializedVersion: 2
>     Hash: 01c155e080f1c19eab7eacdf3723cae7
>   TypeTreeHash:
>     serializedVersion: 2
>     Hash: 55501093163a37cf23c863ea4050548f
> HashAppended: 0
> ClassTypes:
> - Class: 114
>   Script: {fileID: 11500000, guid: 0652a8087db49d248a409593b2b5624b, type: 3}
> - Class: 115
>   Script: {instanceID: 0}
> SerializeReferenceClassIdentifiers: []
> Assets:
> - Assets/MyScriptableObject.asset
> Dependencies: []
> ```

**增量构建是如何工作的？**
在BuildPipeline.BuildAssetBundles()方法的实现中，AssetBundle哈希用于捕获AssetBundle的内容和依赖关系。流程如下：

- 基于当前输入计算AssetBundle哈希值（详见后续部分）
- 如果有一个来自AssetBundle之前构建的.manifest文件，那么就加载哈希值并进行比较。（比较前一版本的.manifest文件和本次构建的哈希）
- 如果哈希值匹配，就计算TypeTree哈希值。
- 如果AssetBundle哈希和TypeTree哈希都匹配，那么不重建AssetBundle（除非指定了ForceRebuildAssetBundle标记）
- 如果构建了AssetBundle，那么新的AssetBundle和TypeTree哈希值会被写入到新生成的.manifest文件中。

这是一个Manifest示例，展示了支持增量构建的数据（如上所示）。

---

> **How is the AssetBundle Hash calculated?**
> The AssetBundle hash is based on the inputs to the AssetBundle build, not the output of the build.This value is calculated by hashing a series of inputs that include:
>
> - TargetPlatform, subtarget
> - Explicitly and implicitly included assets (Specifically the artifactID from the AssetDatabase)
> - Names of the AssetBundles that it depends on
> - Mesh stripping setting
> - Certain BuildAssetBundleOptions
> - For scene bundles: certain global lighting settings (e.g. lightmap mode, fog mode, etc)
> - Shader platform / graphics APIs
> - For bundles with shaders: certain Render Pipeline assets

**AssetBundle哈希是如何被计算出的？**
AssetBundle哈希值基于AssetBundle构建的输入，而不是构建的输出。这个值是通过哈希一系列输入来计算的，包括：

- 目标平台，子平台（如win64/32）
- 明确并隐含包含的资产（具体来说是来自AssetDatabase的artifactID）
- 它所依赖的AssetBundles的名称
- 网格剥离设置
- 某些BuildAssetBundleOptions
- 对于场景bundle：某些全局照明设置（例如光照贴图模式、雾模式等）
- shader平台/图形API
- 对于包含shader的package：某些渲染管线资产

---

> Although a pretty exhaustive calculation, this does not capture every possible influence that can impact the build. Based on bug reports we are aware of some limitations. E.g. Performing the following changes will not change the AssetBundle hash:
>
> - If a asset is moved to a new path (fixed in Unity 2022 and later)
> - If a MonoBehaviour or ScriptableObject keeps the same class name, but moves to a new assembly or namespace
> - If an asset inside a dependent AssetBundle moves to another dependent AssetBundle

尽管这是一项相当详尽的计算，但它并不能捕获所有可能影响构建的因素。基于我们收到的错误报告，我们知道其中有一些限制。比如，执行以下更改将不会改变AssetBundle哈希（做出了以下变更单在增量构建的情况仍不会重新构建，因为AssetHash没有生成新的）：

- 如果资源被移动到新的路径（在Unity 2022及后续版本中已修复）
- 如果一个MonoBehaviour或ScriptableObject保持相同的类名，但移动到了新的程序集或命名空间
- 如果依赖AssetBundle内部的一个资产移动到另一个依赖的AssetBundle

---

> **What is the TypeTreeHash?**
> The Native AssetBundle build implementation uses a second hash value during Incremental build calculations called the **TypeTreeHash**. This value is visible in the .Manifest file, so this section briefly explains how that second value works.
>
> This hash is derived from all the types involved in the AssetBundle. These types are listed in the ClassTypes section of the manifest file, then for each type the hash of the TypeTree is fed into the TypeTreeHash.
>
> For Script types the ClassName, Namespace and Assembly are also hashed.
>
> The purpose of this hash is to detect whether any objects used in the AssetBundle have newer serialization formats. For example adding new fields to a MonoScript or updating to a new version of Unity that changes some built-in objects. A change in serialization format means that the AssetBundle should be regenerated to reflect the latest serialized schema for those objects. Unity does provide its best effort to have backward compatibility to older serialized schemas, e.g. when TypeTrees are included in the AssetBundle, but it is normally best to rebuild content if any type changes for performance and compatibility purposes.
>
> Warning: this TypeTree hash is not part of the AssetBundle hash. So while changes in this hash can force an incremental build, it doesn’t force a change in the overall hash value for the AssetBundle. This is one of the reasons that the AssetBundle hash is not an ideal value for tracking file versions.
>
> This check can be disabled by specifying the BuildAssetBundleOptions.IgnoreTypeTreeChanges flag.

**什么是TypeTree哈希？**
原生AssetBundle构建实现在增量构建计算过程中使用了第二个哈希值，称为**TypeTreeHash**。

这个值可以在.Manifest文件中看到，所以这一节简要解释一下这第二个值是如何运作的。这个哈希值是从AssetBundle涉及的所有类型中得出的。这些类型在manifest文件的ClassTypes部分中列出，然后对于每种类型，都将TypeTree的哈希值计入TypeTree哈希值。

对于脚本类型，还会对类名、命名空间和程序集进行哈希。

这个哈希值的目的是检测在AssetBundle中使用的任何对象是否有较新的序列化格式。例如向MonoScript添加新字段或更新到改变了某些内置对象的Unity新版本。序列化格式的改变意味着应当重新生成AssetBundle，以反映这些对象最新的序列化情况。Unity尽其最大努力提供对旧序列化方案的向后兼容性，例如当TypeTrees包含在AssetBundle中时，但通常最好是如果任何类型改变，为了性能和兼容性，都对内容进行重建。

警告：这个TypeTree哈希值不是AssetBundle哈希值的一部分。所以虽然这个哈希值的改变可以强制进行增量构建，但它并不强制改变AssetBundle的整体哈希值。这也是AssetBundle哈希值不是跟踪文件版本的理想值的原因之一。

通过指定BuildAssetBundleOptions.IgnoreTypeTreeChanges标志，可以禁用这个检查。

---

> **Can the known limitations be fixed?**
> The cases of incomplete hashing can have serious impact, with the potential of null references, crashes or other unexpected failures on end user devices. That is because an older AssetBundle, with out-of-date content, might have the same hash as a newly built AssetBundle that has correct content. We can call this a “hash conflict”.
>
> As mentioned above we are aware of some limitations in the hashing algorithm, so a logical step would be to fix these known limitations.
>
> However, because the input calculation and the visible Hash are the same thing, there are backward compatibility challenges for improving the incremental build calculation.
>
> For example, if Unity starts to incorporate more information about script types into the hash, then this would change the hash for all existing AssetBundles that have MonoBehaviours and ScriptableObjects. Existing projects that do a minor upgrade of Unity version might suddenly see all their stable AssetBundles requiring a new build, even when the resulting content is actually unchanged. Rebuilding can take a long time, and deploying new AssetBundles can result in large usage of bandwidth or excessive downloads to devices. So we try to keep things quite stable in the area of AssetBundles on our Long Term Support versions of Unity. Because of that concern, fixes to the Incremental build calculation are not normally backported, and require the introduction of new flags.
>
> For new releases of Unity we are able to improve the AssetBundle pipeline code and reduce these limitations. That is because AssetBundle content will practically always change, at least a little bit, when doing a major upgrade of Unity for an existing project, so it is a good opportunity to introduce code changes that effectively force a clean build.

不完全的哈希可能会产生严重的影响，可能会在最终用户设备上导致空引用，崩溃或其他意外故障。这是因为一个旧的AssetBundle，其内容已过时，可能与新建的、内容正确的AssetBundle具有相同的哈希。我们可以称之为“哈希冲突”。

如上所述，我们已经知道哈希算法中存在一些限制，所以一个逻辑步骤就是修复这些已知的限制。

然而，因为输入计算和可见的哈希是同一件事，所以改进增量构建计算存在向后兼容性的挑战。

例如，如果Unity开始将更多的关于脚本类型的信息纳入哈希，那么这将会改变所有已经含有MonoBehaviours和ScriptableObjects的现有AssetBundles的哈希。当现有的项目只进行Unity版本的小升级时，他们可能会突然发现他们的所有稳定的AssetBundles都需要进行新的构建，即使结果的内容实际上没有改变。重构可能需要很长时间，部署新的AssetBundles可能会导致大量的带宽使用或者过量的下载到设备。因此，我们尽力在Unity的长期支持版本的AssetBundles区域保持稳定。由于这个问题，增量构建计算的修复通常不会被回溯，而且需要引入新的标志。

对于新版本的Unity，我们可以优化AssetBundle管道代码并减少这些限制。这是因为AssetBundle的内容在对现有项目进行Unity的重大升级时，几乎总是会改变，至少有一点小的改变，所以这是一个很好的机会去引入那些实际上强制进行清理构建的代码变化。

---

> **Clean Builds**
> The risk of incremental builds is that Unity might decide that an AssetBundle from a previous build is valid, based on all the checks that it performs as it calculates the AssetBundle hash and TypeTree hash. Because those checks are not 100% exhaustive then it may leave an AssetBundle alone that would actually have different content if it had been rebuilt.
>
> The ForceRebuildAssetBundle flag can be used to force each AssetBundle to rebuild, even if the input hash and typeree hash have not changed. Because incremental builds rely on the .manifest files then it is possible to force the rebuild of AssetBundles by erasing the build folder. In fact, erasing the output folder prior to a build can be a good approach, to clear out any obsolete or renamed AssetBundles prior to a fresh build.
>
> Of course the downside of clean builds is the performance cost of repeating unnecessary build work, potentially adding many hours to the build time. In more advanced situations, where users have a very precise idea what which AssetBundles need to be rebuilt, then it could be feasible to erase individual .manifest files, instead of using ForceRebuildAssetBundle . That would be a way to force certain AssetBundles to rebuild while leaving others that have predictable content to be handled by the regular Incremental Build calculation.
>
> Incremental builds may make sense for internal builds, e.g. testing builds during production, to help with iteration time. The risk of a hash conflict can exist but with less serious impact. And in fact hash conflicts can be detected with some extra code running as part of the build script.
>
> Doing a clean build doesn’t prevent the possibility that multiple versions of an AssetBundle can have the same AssetBundle hash, instead it just forces the “correct” current version is generated.

**全新构建**
增量构建的风险在于Unity可能会判断出一个来自之前构建的AssetBundle是有效的，基于它在计算AssetBundle哈希和TypeTree哈希的过程中执行的所有检查。因为这些检查并非100%完备，所以它可能会保留一个实际上如果被重建会有不同内容的AssetBundle。

ForceRebuildAssetBundle标记可以用来强制每个AssetBundle重建，即使输入哈希和typeree哈希没有改变。因为增量构建依赖于.manifest文件，所以我们可以通过删除构建文件夹来强制重建AssetBundles。实际上，在构建之前清理输出文件夹可能是一个不错的方法，可以清理出任何过时的或重命名的AssetBundles，以便进行全新的构建。

当然，全新构建的缺点在于重复不必要的构建工作的性能开销，可能会为构建时间增加许多小时。在更高级的场景中，用户可能对需要重新构建哪些AssetBundle有非常精确的了解，那么可能可以考虑删除单个.manifest文件，而不是使用ForceRebuildAssetBundle。这将是强制某些AssetBundle进行重建的方式，同时让那些内容可预知的其他AssetBundle由常规的增量构建计算来处理。

在内部构建中，例如在生产过程中进行测试构建，增量构建是可能有意义的，以帮助缩短迭代时间。哈希冲突的风险是存在的，但影响较小。实际上，可以通过在构建脚本中运行一些额外代码来检测哈希冲突。

进行全新构建并不能阻止一个AssetBundle的多个版本可能有相同的AssetBundle哈希，而只是强制生成“正确”的当前版本。

---

> **Overview of UnityWebRequestAssetBundle and the AssetBundle Cache**
> The [UnityWebRequestAssetBundle ](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/Networking.UnityWebRequestAssetBundle.html)API makes it easy to incorporate AssetBundle downloading into a player build, especially because it is available on all supported platforms. On most platforms this includes caching support.
>
> In order to use the AssetBundle cache a version must be specified, otherwise the same bundle can be downloaded over and over again, every time it is requested.
>
> It is up to the user to provide any 128-bit (hash) or 32-bit (uint) value they like to distinguish the “version” of the AssetBundle. This could be the hash value calculated by the AssetBundle build, or a regular numeric version number (1,2,3…) or some other value that fits into the customer’s build and release system.
>
> For example the second argument to this signature is the version hash:

```c#
UnityWebRequest UnityWebRequestAssetBundle.GetAssetBundle(Uri uri, Hash128 hash, uint crc);
```

> The specified version is recorded in the cache, along with the downloaded AssetBundle.
>
> If, at a later time, the code attempts to download the same bundle again, and specify the exact same version hash, then the cached version will be reused, rather than a new download. If the version hash does not match then the AssetBundle is downloaded again, and becomes the newly cached version.
>
> Note - this check is simply checking whether the provided 128-bit value matches the 128-bit value when the AssetBundle was put into the cache, there is no hashing performed at that point.
>
> So, to successfully use this design, it is important to update the version when a new AssetBundle build is released for download. That way devices will download and use the newer version instead of a previous version that might be cached locally.
>
> Note: If a downloaded file is using LZMA format (which is the default) then it is recompressed on the device to LZ4 and put into the cache. The AssetBundles can also be cached in uncompressed format by setting Caching.enableCompression to false. These compression transformations can have implications if the full file hash is being used as a unique AssetBundle version identifier.

**UnityWebRequestAssetBundle和AssetBundle缓存的概述**
[UnityWebRequestAssetBundle](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/Networking.UnityWebRequestAssetBundle.html) API使得将AssetBundle下载整合到玩家构建中变得简单，尤其是因为它在所有支持的平台上都可用。在大多数平台上，这包括了缓存支持。

为了使用AssetBundle缓存，必须指定一个版本，否则每次请求时都可能重复下载相同的bundle。

用户可以提供任何他们喜欢的128位（哈希）或32位（uint）的值来区分AssetBundle的“版本”。这可以是通过AssetBundle构建计算的哈希值，或是一个常规的数字版本号（1，2，3...），或是其他能够适应客户端的构建和发布系统的值。

例如，这个签名的第二个参数就是版本哈希。

```c#
UnityWebRequest UnityWebRequestAssetBundle.GetAssetBundle(Uri uri, Hash128 hash, uint crc);
```

指定的版本会与下载的AssetBundle一起记录在缓存中。

如果，稍后代码试图再次下载同一个bundle，并指定完全相同的版本哈希，那么将会重用缓存的版本，而不是重新下载。如果版本哈希不匹配，那么AssetBundle将会再次被下载，并成为新的缓存版本。

注意 - 这个检查只是简单地检查提供的128位数值是否与将AssetBundle放入缓存时的128位数值匹配，此时并没有进行哈希计算。因此，为了成功使用这个设计，更新新的AssetBundle构建发布下载的版本是重要的。

这样，设备将会下载并使用新版本，而不是可能在本地缓存的旧版本。

注意：如果下载的文件使用的是LZMA格式（这是默认设置）那么它会在设备上被重新压缩为LZ4并放入缓存。通过将Caching.enableCompression设置为false，AssetBundles也可以以未压缩格式缓存。这些压缩转换可能会影响到如果正在将完整文件哈希用作唯一的AssetBundle版本标识符的话。

---

> **Should the AssetBundle Hash be used as a version?**
> It can be tempting to use the AssetBundle hash reported in the Manifest files as a version hash for the AssetBundle cache. For example a user may distribute the Manifest AssetBundle after doing a new build, and then run code in the player to enumerate AssetBundles and get their hashes, e.g. with [AssetBundleManifest.GetAssetBundleHash](https://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAssetBundleHash.html). That hash might then be provided when calling GetAssetBundle() as a way to enable caching.
>
> However, as mentioned previously, there are limitations of the AssetBundle hash. So it is possible that an AssetBundle is rebuilt with new content, but the AssetBundle hash is exactly the same, which is a “hash conflict”. That means that a device may have an older, incompatible version of the AssetBundle cached locally, and it will be stuck using the old one instead of downloading the new one because it thinks it already has that version. This could show up as unexpected behavior, like missing content or crashes on end user devices, that are not reproduced when using a fresh install.
>
> Recovery from such a situation might require some extra coding. If there are two bundles in circulation with the same version hash then the CRC can be useful. There is support in the AssetBundle.LoadFromFile() API to check the CRC and fail the load if the content doesn’t match the expected CRC. This would detect an incompatible AssetBundle, so that it can be discarded. However, doing an CRC check on each AssetBundle load can really slow things down, so normally we only recommend it for use with UnityWebRequestAssetBundle.GetAssetBundle(). That API only checks the CRC at the time of download, which is efficient but doesn’t help if an AssetBundle with the wrong CRC is already cached! Users could potentially write their own code to check CRCs more efficiently, e.g. only checking CRC of cached AssetBundles when a new release of a game has occurred. The Caching API can be used to enumerate the cache and clear individual items.
>
> It is also possible to detect hashing conflicts at the time of the build, and avoid releasing a new bundle if its hash has not changed. Resolving this situation might require renaming bundles or making small changes to the content to bypass the problem.

**应该将AssetBundle哈希用作版本吗？**
使用Manifest文件中报告的AssetBundle哈希作为AssetBundle缓存的版本哈希可能是一种诱人的选择。例如，用户可能在完成新的构建后分发Manifest AssetBundle，然后在玩家中运行代码来枚举AssetBundles并获取他们的哈希，例如，使用[AssetBundleManifest.GetAssetBundleHash](https://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAssetBundleHash.html)。然后，可能会在调用GetAssetBundle()时提供该哈希，作为启用缓存的方式。

然而，如前所述，AssetBundle哈希存在一些限制。因此，有可能AssetBundle被用新的内容重新构建，但是AssetBundle哈希却完全相同，这就是一个“哈希冲突”。这意味着一个设备可能在本地缓存了一个较老，不兼容的AssetBundle版本，而且它会因为认为已经拥有了那个版本而坚持使用旧的版本，而不去下载新的版本。这可能会表现为意外的行为，如在最终用户设备上出现的丢失内容或崩溃，这些问题在使用全新安装时无法复现。

从这种情况恢复可能需要一些额外的编码。如果有两个拥有相同版本哈希的bundle在流通，则CRC可能会有用。AssetBundle.LoadFromFile() API支持检查CRC并在内容与预期的CRC不匹配时失败加载。这将会侦测到不兼容的AssetBundle，以便可以丢弃它。然而，对每个AssetBundle加载进行CRC检查可能会大大降低速度，所以我们通常只推荐在UnityWebRequestAssetBundle.GetAssetBundle()中使用它。这个API只在下载时检查CRC，这很高效，但是如果一个错误CRC的AssetBundle已经被缓存，它就无济于事了！用户可能可以编写自己的代码来更有效地检查CRC，例如，仅当游戏的新版本发布时检查缓存的AssetBundles的CRC。Caching API可以用于列举缓存并清理单个项目。

在构建时也可以检测到哈希冲突，并且如果它的哈希没有改变则可以避免释放新的bundle。解决这种情况可能需要重命名bundles或对内容进行小的更改以规避问题。

---

> **Alternative Version Approaches**
> Rather than facing the challenge of recovering from a “hash conflict”, it seems better to use a more unique value as an AssetBundle version.
>
> For example the CRC itself can serve as a version identifier (cast into a 128 byte value). Because it is calculated based on the uncompressed content it is resilient to compression changes, and it is based on the actual built content, so does not have the flaws of the Native AssetBundle hash calculation. The .manifest file itself could be hashed, because it contains the CRC along with other distinguishing content. Or the build pipeline of a production may have its own version counting or time stamp available that can serve as a unique version identifier.
>
> It is also possible to hash the bytes of the AssetBundle file. Hashing file content is a common and robust way to assign a unique version identifier to a file. However when using this approach it is recommended to use the [BuildAssetBundleOptions.AssetBundleStripUnityVersion](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/BuildAssetBundleOptions.AssetBundleStripUnityVersion.html) flag when building the AssetBundles. And it is important to be aware of the compression changes that can occur if AssetBundles are built with LZMA, instead of the compression used in the Cache (LZ4 or Uncompressed), because any recompression will change the file’s content.
>
> No matter what method is used, there needs to be a mechanism to distribute these versions as a new build is uploaded. This could be a simple JSON file that is generated as a post-build step, and the player build will download this file as a first step of checking for updated AssetBundles.

**替代版本方法**
与其面临从“哈希冲突”恢复的挑战，不如使用一个更独特的值作为AssetBundle的版本。

例如，CRC本身就可以作为版本标识符（转换为128字节值）。因为它是基于未压缩的内容计算的，所以能够承受压缩的变化，它是基于实际构建的内容的，所以决无Native AssetBundle哈希计算的缺陷。.manifest文件本身可以被哈希，因为它包含了CRC以及其他区别的内容。或者，生产的构建管道可能具有可作为唯一版本标识符的自己的版本计数或时间戳。

也可以对AssetBundle文件的字节进行哈希。哈希文件内容是给文件分配一个唯一版本标识符的常见和稳健的方式。然而，在使用这种方法时，建议在构建AssetBundles时使用[BuildAssetBundleOptions.AssetBundleStripUnityVersion](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/BuildAssetBundleOptions.AssetBundleStripUnityVersion.html)标志。并且，要注意如果AssetBundles是用LZMA构建的而不是在缓存中使用的压缩（LZ4或未压缩），那么可能发生的压缩改变，因为任何重新压缩都会改变文件的内容。

无论使用哪种方法，都需要一个机制在新的构建上传时分发这些版本。这可能是一个在构建后生成的简单的JSON文件，玩家构建将会下载这个文件作为检查更新的AssetBundles的第一步。

---

> **Update: BuildAssetBundleOptions.UseContentHash**
>
> Starting with 2022.3.8f1 we have introduced a new flag, BuildAssetBundleOptions.UseContentHash. When specified Unity will use the content for the AssetBundle hash, instead of the input hash described previously. The decision for whether to rebuild an AssetBundle is still dependent on the input hash, so that will also be tracked as an additional value in the .manifest file. Using the flag means the AssetBundle hash is safe to use for UnityWebRequestAssetBundle and "hash conflicts" should not occur.
>
> There can still be bugs in edge cases that impact incremental builds, where an AssetBundle does not rebuild even though some input that influences the build results has changed. So it remains a recommendation to use clean builds for official releases.
>
> It is strongly recommended to also use the BuildAssetBundleOptions.AssetBundleStripUnityVersion flag when UseContentHash is use, so that the content is not changed after minor Unity upgrades.
>
> Note also the content-based AssetBundle hash cannot be calculated without actually building the AssetBundle. So it is no longer available when BuildAssetBundleOptions.DryRunBuild is specified.
>
> **Other AssetBundle APIs**
> The Scriptable Build Pipeline (used by Addressables) and the [Multi-process form of BuildPipeline.BuildAssetBundles](https://docs.unity3d.com/2023.1/Documentation/Manual/Build-MultiProcess.html) (introduced in 2023.1) use content inside the bundle to calculate the AssetBundle hash. That makes them much more resilient to the risk of a “hash conflict”, and safer to use with UnityWebRequestAssetBundle.GetAssetBundle().
>
> The [BuildAssetBundleOptions.AssetBundleStripUnityVersion](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/BuildAssetBundleOptions.AssetBundleStripUnityVersion.html) flag is recommended (or the equivalent flag in Addressables) so that doing a minor upgrade in Unity does not force a change to the AssetBundle hash.
>
> Doing a clean build for official releases is always recommended, including when using Addressables. While our newer approaches have better input calculations, there still can be some cases where a global setting, build callback or other factor can influence the content of the AssetBundle in a way that is not predicted by the incremental build calculation.
>
> **Conclusion**
> Hopefully this deep dive into the details of the AssetBundle incremental build support is helpful for managing AssetBundles using BuildPipeline.BuildAssetBundles(). The implementations available through Addressables and improvements for 2023 have addressed the risk of “hash conflicts”. And BuildAssetBundleOptions.UseContentHash is available in more recent versions of 2022. But older versions of Unity and BuildPipeline.BuildAssetBundles() are still widely used. This older API has been successfully used by many projects to deploy content, but being aware of the risk of hash conflicts can help avoid some potential pitfalls.
>
> We also hope that, by posting this to the forum, the community will chime in and share some techniques and best practices for dealing with Incremental Builds and the AssetBundle cache.

**更新：BuildAssetBundleOptions.UseContentHash**

从2022.3.8f1开始，我们引入了一个新的标志，BuildAssetBundleOptions.UseContentHash。当指定该标志时，Unity 将使用 AssetBundle 的内容进行哈希计算，而不是之前描述的输入哈希。是否重新构建 AssetBundle 的决定仍然依赖于输入哈希，因此它也将被作为 .manifest 文件中的一个额外值进行跟踪。使用该标志意味着 AssetBundle 哈希可以安全地用于 UnityWebRequestAssetBundle，且"哈希冲突"不应发生。

仍然可能存在影响增量构建的边缘情况的 bug，其中 AssetBundle 尽管有一些影响构建结果的输入发生了变化，但并未重新构建。因此，我们仍然建议对官方发布使用干净的构建。

强烈建议在使用 UseContentHash 时也使用 BuildAssetBundleOptions.AssetBundleStripUnityVersion 标志，这样在 Unity 升级后内容不会发生变化。

还需注意，基于内容的 AssetBundle 哈希无法在实际构建 AssetBundle 之前计算。因此，当指定 BuildAssetBundleOptions.DryRunBuild 时，它将不再可用。

**其他 AssetBundle APIs**
Scriptable Build Pipeline（由 Addressables 使用）和 [Multi-process form of BuildPipeline.BuildAssetBundles](https://docs.unity3d.com/2023.1/Documentation/Manual/Build-MultiProcess.html)（在 2023.1 中引入）使用包内的内容来计算 AssetBundle 的哈希。这使得它们对“哈希冲突”的风险更具有弹性，可以安全地与 UnityWebRequestAssetBundle.GetAssetBundle() 一起使用。

建议使用 [BuildAssetBundleOptions.AssetBundleStripUnityVersion](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/BuildAssetBundleOptions.AssetBundleStripUnityVersion.html) 标志（或 Addressables 中的等效标志），以便在 Unity 中进行小幅升级时不会强制改变 AssetBundle 的哈希。

对于官方发布，始终建议进行干净的构建，包括在使用 Addressables 时。虽然我们的新方法有更好的输入计算，但仍然可能有一些情况，其中全局设置、构建回调或其他因素可能以未按增量构建计算预测的方式影响 AssetBundle 的内容。

**结论**
希望这个关于 AssetBundle 增量构建支持的详尽讨论对于使用 BuildPipeline.BuildAssetBundles() 管理 AssetBundles 有所帮助。通过 Addressables 可用的实现和 2023 年的改进解决了“哈希冲突”的风险。并且 BuildAssetBundleOptions.UseContentHash 可用在 2022 年更近期的版本。但是旧版本的 Unity 和 BuildPipeline.BuildAssetBundles() 都还在广泛使用。虽然许多项目已经成功使用这个旧的 API 来部署内容，但是了解哈希冲突的风险可以帮助避免一些潜在的陷阱。

们也希望，通过将此发表在论坛上，社区将参与其中，并分享一些处理增量构建和 AssetBundle 缓存的技巧和最佳实践。

---

> I've also prepared a demonstration script to help make the rather technical text a bit more concrete.
>
> It has only had some mild testing, so consider that there are lots of disclaimers here that putting it into production is at your own risk. But if you detect any bugs or make improvements I would like to hear about it in this thread, and I hope it can be very helpful.

我也准备了一个演示脚本，以帮助让这些较为技术性的文本变得更加具体。这个脚本只进行了一些轻度的测试，所以请注意这里有很多免责声明，将它投入生产环境是你自己的风险。但是，如果你发现任何错误或者进行了改进，我希望可以在这个帖子中听到你的反馈，我希望它能够带给你很大的帮助。

```c#
using System;
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using UnityEditor;
using UnityEngine;
 
// 这是一个示例代码，展示了如何利用现有的Unity和System API来监控AssetBundle增量构建的输出，
// 包括检测"哈希冲突"。该代码已经在2021.3版本中进行了测试，如果发现任何问题，请在发布此帖的Unity论坛中报告。
// 如果觉得有用，你可以随意将其整合到你自己的构建脚本中。
// 使用方法：根据你自己的构建布局更新脚本。然后运行菜单项 "Test Repro/Build Asset Bundles"。
// 输出信息会被写入到Console窗口。
 
public class BuildBundles
{
    static AssetBundleBuild[] GetBundleDefinitions()
    {
        // Adjust this method according to your own Assets and desired build layout
 
#if TRUE
        AssetBundleBuild[] bundleDefinitions = new AssetBundleBuild[2];
 
        bundleDefinitions[0].assetBundleName = "bundle_prefab";
        bundleDefinitions[0].assetNames = new string[]
        {
            "Assets/MyPrefab.prefab",
        };
 
        bundleDefinitions[1].assetBundleName = "bundle_sobject";
        bundleDefinitions[1].assetNames = new string[]
        {
            "Assets/MyScriptableObject.Asset"
        };
 
        return bundleDefinitions;
#else
        //这个调用可以用来构建在Inspector/AssetDatabase定义的AssetBundles。
        return ContentBuildInterface.GenerateAssetBundleBuilds();
#endif
    }
 
    [MenuItem("Test Repro/Build Asset Bundles")]
    static void BuildAssetBundles()
    {
        string buildPath = Application.streamingAssetsPath;
 
        // 构建AssetBundles
        // 为了测试，你可以试验再次调用它，例如，在修改了ScriptableObject代码后，看看哪些部分重新构建了
        // 每次包改变时，都应进行一次新的玩家构建
 
        var bundleDefinitions = GetBundleDefinitions();
        var reporter = new IncrementalBuildReporter(buildPath);
        reporter.ReportToConsole();
 
        Directory.CreateDirectory(buildPath);
        
        //注意：BuildAssetBundleOptions.ForceRebuildAssetBundle没有被显式指定，所以可以查看增量构建的结果
        var manifest = BuildPipeline.BuildAssetBundles(buildPath,
            bundleDefinitions,
            BuildAssetBundleOptions.AssetBundleStripUnityVersion,
            BuildTarget.StandaloneWindows64);
 
        if (manifest != null)
        {
            reporter.DetectBuildResults();
            reporter.ReportToConsole();
        }
        else
        {
            Debug.Log("Build failed");
        }
    }
}
 
struct BundleBuildInfo
{
    public Hash128 bundleHash; // 这俩都由unity计算
    public uint crc;           // 
    public DateTime timeStamp; //可以用于检测bundle是否被重建或使用了前一版本的文件（bundle）
    public string contentHash; // MD5, 在CRC发生改变时改变
 
    public override string ToString()
    {
        return $"Unity hash: {bundleHash} Content MD5: {contentHash} CRC: {crc.ToString("X8")} Write time: {timeStamp}";
    }
}
 
public class IncrementalBuildReporter
{
    Dictionary<string, BundleBuildInfo> m_previousBuildInfo; // map from path to BuildInfo
    string m_buildPath; // Typically a relative path within the Unity project
    string m_manifestAssetBundlePath;
 
    StringBuilder m_report;
 
    public IncrementalBuildReporter(string buildPath)
    {
        m_report = new StringBuilder();
        m_buildPath = buildPath;
        m_previousBuildInfo = new();
 
        var directoryName = Path.GetFileName(buildPath);
 
        // Special AssetBundle that stores the AssetBundleManifest follows this naming convention
        m_manifestAssetBundlePath = buildPath + "/" + directoryName;
 
        if (!File.Exists(m_manifestAssetBundlePath))
        {
            // Expected on the first build
            m_report.AppendLine("No Previous Build Found");
            return;
        }
 
        m_report.AppendLine("Collecting info from previous build");
        CollectBuildInfo(m_previousBuildInfo);
    }
 
    public void ReportToConsole()
    {
        Debug.Log(m_report.ToString());
        m_report.Clear();
    }
 
    private void CollectBuildInfo(Dictionary<string, BundleBuildInfo> bundleInfos)
    {
        var manifestAssetBundle = AssetBundle.LoadFromFile(m_manifestAssetBundlePath);
        try
        {
            var assetBundleManifest = manifestAssetBundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");
 
            var allBundles = assetBundleManifest.GetAllAssetBundles();
            foreach (var bundleRelativePath in allBundles)
            {
                // bundle is the AssetBundle's path relative to the root build folder
                var bundlePath = m_buildPath + "/" + bundleRelativePath;
                if (!File.Exists(bundlePath))
                {
                    // Bundles may have been manually erased or moved after the build
                    m_report.AppendLine("AssetBundle " + bundlePath + " is missing from disk");
                    continue;
                }
 
                // Get the CRC from the bundle's .manifest file
                if (!BuildPipeline.GetCRCForAssetBundle(bundlePath, out uint crc))
                {
                    m_report.AppendLine("Failed to read CRC from manifest file of " + bundlePath);
                    continue;
                }
 
                var fileInfo = new FileInfo(bundlePath);
                var bundleInfo = new BundleBuildInfo()
                {
                    bundleHash = assetBundleManifest.GetAssetBundleHash(bundleRelativePath), //
                    crc = crc,
                    timeStamp = fileInfo.CreationTime,
                    contentHash = GetMD5HashFromAssetBundle(bundlePath)
                };
 
                bundleInfos.Add(bundleRelativePath, bundleInfo);
            }
        }
        finally
        {
            manifestAssetBundle.Unload(true);
        }
    }
 
    public void DetectBuildResults()
    {
        m_report.AppendLine().AppendLine("Collecting results of new Build:");
 
        var newBuildInfo = new Dictionary<string, BundleBuildInfo>();
        CollectBuildInfo(newBuildInfo);
 
        foreach (KeyValuePair<string, BundleBuildInfo> dictionaryEntry in newBuildInfo)
        {
            string bundlePath = dictionaryEntry.Key;
            BundleBuildInfo newBundleInfo = dictionaryEntry.Value;
 
            if (m_previousBuildInfo.TryGetValue(bundlePath, out BundleBuildInfo previousBundleInfo))
            {
                if (previousBundleInfo.timeStamp == newBundleInfo.timeStamp)
                {
                    // Bundle was not rebuilt.  Do some sanity checking just in case the timestamp is misleading
                    if (previousBundleInfo.crc != newBundleInfo.crc ||
                        previousBundleInfo.contentHash != newBundleInfo.contentHash)
                    {
                        m_report.AppendLine($"*UNEXPECTED* [Timestamp match with new content]: {bundlePath}\n\tNow:  {newBundleInfo} \n\tWas: {previousBundleInfo}");
                    }
                    else
                    {
                        // Incremental build decided not to build this bundle
                        m_report.AppendLine($"[Not rebuilt]: {bundlePath}\n\t{newBundleInfo}");
                    }
                }
                else if (previousBundleInfo.bundleHash == newBundleInfo.bundleHash)
                {
                    if (previousBundleInfo.crc != newBundleInfo.crc)
                    {
                        // Hash is the same, but according to the CRC, the bundle has new content, so this is a "hash conflict".
                        // This is problematic if the hash is used to distinguish different versions of the AssetBundle (e.g. along with the AssetBundle cache)
                        // If this occurs be wary of releasing this build.
                        m_report.AppendLine($"*WARNING* [New CRC content, but unchanged hash]: {bundlePath}\n\tNow: {newBundleInfo}\n\tWas: {previousBundleInfo}");
                    }
                    else if (previousBundleInfo.contentHash != newBundleInfo.contentHash)
                    {
                        // Normally shouldn't happen, because the CRC check above should also trigger
                        m_report.AppendLine($"*WARNING* [New file content, unchanged hash]: {bundlePath}");
                    }
                    else
                    {
                        // Expected with ForceRebuildAssetBundle or if Unity is being conservative and rebuilding something that might have changed
                        m_report.AppendLine($"[Rebuilt, identical content]: {bundlePath}\n\tNow: {newBundleInfo}\n\tWas: {previousBundleInfo}");
                    }
                }
                else
                {
                    if (previousBundleInfo.contentHash == newBundleInfo.contentHash)
                    {
                        // Expected if the incremental build heuristic has changed, e.g. when upgrading Unity
                        m_report.AppendLine($"[Rebuilt, new hash produced identical content]:{bundlePath}\n\tNow: {newBundleInfo}\n\tWas: {previousBundleInfo}");
                    }
                    else
                    {
                        // The normal case for a AssetBundle that required rebuild
                        m_report.AppendLine($"[Rebuilt, new content]: {bundlePath}\n\tNow: {newBundleInfo}\n\tWas: {previousBundleInfo}");
                    }
                }
 
                // Clear it out so that we can detect obsolete AssetBundles
                m_previousBuildInfo.Remove(bundlePath);
            }
            else
            {
                m_report.AppendLine($"[Brand new]: {bundlePath}\n\t{newBundleInfo}");
            }
        }
 
        // 此结构中剩余的任何内容都没有与新构建的AssetBundle匹配（因此可能可以被擦除）。
        foreach (KeyValuePair<string, BundleBuildInfo> dictionaryEntry in m_previousBuildInfo)
        {
            m_report.AppendLine($"[Obsolete bundle]: {dictionaryEntry.Key}\n\t{dictionaryEntry.Value}");
        }
    }
 
    private static string GetMD5HashFromAssetBundle(string fileName)
    {
        // 注意：如果压缩方式变动（例如，为了AssetBundle缓存做的LZMA -> LZ4转换），文件内容将会改变
        // 提示：若你使用哈希来追踪AssetBundle的版本，那么我们推荐使用AssetBundleStripUnityVersion标志。
        FileStream file = new FileStream(fileName, FileMode.Open);
 
        var md5 = MD5.Create();
        byte[] hash = md5.ComputeHash(file);
        file.Close();
 
        // Convert to string
        var sb = new StringBuilder();
        for (int i = 0; i < hash.Length; i++)
            sb.Append(hash[i].ToString("x2"));
        return sb.ToString();
    }
}
```

