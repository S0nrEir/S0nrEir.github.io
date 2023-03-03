---
title: [译]Unity博客《custom == operator,should we keep it?》
tags: ["原理"]
---



这是一篇解释了“Unity为什么要使用自定义的==运算符，以及这样做导致的结果”的博客。

简译顺便学习。

原文地址：[**Custom == operator, should we keep it?**](https://blog.unity.com/technology/custom-operator-should-we-keep-it)

When you do this in Unity:

```c#
if (myGameObject == null) {}
```

*Unity does something special with the == operator. Instead of what most people would expect, we have a special implementation of the == operator.*

*This serves two purposes:*

1) *When a MonoBehaviour has fields, in the editor only**[1]**, we do not set those fields to "real null", but to a "fake null" object. Our custom == operator is able to check if something is one of these fake null objects, and behaves accordingly. While this is an exotic setup, it allows us to store information in the fake null object that gives you more contextual information when you invoke a method on it, or when you ask the object for a property. Without this trick, you would only get a NullReferenceException, a stack trace, but you would have no idea which GameObject had the MonoBehaviour that had the field that was null. With this trick, we can highlight the GameObject in the inspector, and can also give you more direction: "looks like you are accessing a non initialised field in this MonoBehaviour over here, use the inspector to make the field point to something".*

2) *purpose two is a little bit more complicated.*

   *When you get a c# object of type "GameObject"**[2]**, it contains almost nothing. this is because Unity is a C/C++ engine. All the actual information about this GameObject (its name, the list of components it has, its HideFlags, etc) lives in the c++ side. The only thing that the c# object has is a pointer to the native object. We call these c# objects "wrapper objects". The lifetime of these c++ objects like GameObject and everything else that derives from UnityEngine.Object is explicitly managed. These objects get destroyed when you load a new scene. Or when you call `Object.Destroy(myObject);` on them. Lifetime of c# objects gets managed the c# way, with a garbage collector. This means that it's possible to have a c# wrapper object that still exists, that wraps a c++ object that has already been destroyed. If you compare this object to null, our custom == operator will return "true" in this case, even though the actual c# variable is in reality not really null.*

当你在Unity中这样做：

```c#
if (myGameObject == null) {}
```

Unity对==操作符做了一些特殊的事，并不如人们期待的那样，我们对==操作符有特殊实现。

这主要出于两个目的：

1. 当一个Monobehaviour拥有字段时，仅在编辑器模式下（后面会解释），我们并不真的将这些字段设置为“真的null”，而是一个“假null”对象。我们自定义的==运算符会检查某些东西是否为这些“fake null objects”的一部分（指fields之类的），并进行相应处理。虽然这样做很奇怪，它允许我们在fake null object上存储信息，当你调用其上的方法时或访问它的某个属性时以给你更多的上下文信息。如果没有这项技巧，你只会得到一个NullReferenceException，和一个调用栈追踪，但你可能不知道哪个GameObject上有空字段的Monobehavior。有了这项技巧，我们可以在inpector面板中高亮（出问题的）GameObject，并且也可以给你更多引导（信息）：“看起来你正好在访问这个MonoBehavior上一个没有初始化的字段，使用inspector来让字段指向某个东西（初始化）”。（大意就是为了在编辑器调试和出错时能够给出开发者更多信息，方便debug）。

第二个目的有点复杂。

2. **当你得到一个类型为“GameObject”的C#对象时，它几乎不包含任何东西。这是因为Unity是一个C/C++引擎。所有关于这个GameObject的真实的信息（比如name,持有的components,HideFlags之类的属性）都存在于（引擎的）C++层。这个C#对象唯一持有的是指向这个原生对象（C++对象）的指针。我们管这些C#对象叫做“包装对象”。像GameObject或者所有其他派生自UnityEngine.Object的C++对象的生命周期都是显式管理的。当你加载一个新场景或调用Oblject.Destroy(myObject)时这些对象会被销毁。而C#对象的生命周期则通过C#的方式，由GC来管理。这意味着可能C#的包装对象仍然存在，但它包裹（指向）的C++对象已经被销毁了。如果你把这个C#对象和null进行比较的话，结果会返回true，即使真实的C#对象并不是真的null**

------

*While these two use cases are pretty reasonable, the custom null check also comes with a bunch of downsides.*

这两种理由都是很合理的，但自定义的判空检查也带来了一些问题。

------

- *It is counterintuitive.*
- *Comparing two UnityEngine.Objects to eachother or to null is slower than you'd expect.*
- *The custom ==operator is not thread safe, so you cannot compare objects off the main thread. (this one we could fix).*
- *It behaves inconsistently with the ?? operator, which also does a null check, but that one does a pure c# null check, and cannot be bypassed to call our custom null check.*

- 它是反直觉的。
- 让两个派生自UnityEngine.Obejcts的对象互相比较或是让他们检查是否为空，比预想的要慢（指性能方面）。
- 自定义的==运算符并非线程安全的，你不能在主线程之外对两个对象进行比较（我们会修复这个）。
- 它与??运算符的表现不一致，??运算符也是做空检查的，但它是纯C#的判空检查，并且无法被绕过以调用自定义的判空检查。（??运算符和自定义的==运算符逻辑不一致）

------

*Going over all these upsides and downsides, if we were building our API from scratch, we would have chosen not to do a custom null check, but instead have a myObject.destroyed property you can use to check if the object is dead or not, and just live with the fact that we can no longer give better error messages in case you do invoke a function on a field that is null.*

综合考虑这些问题，如果我们从头构建API，可能不会选择使用自定义的判空检查，而是使用myObject.destroyed这样的属性取而代之这样你就可以检查对象是否已经销毁，并且接受一个事实：当你调用一个空字段上的方法时，Unity没办法给你更好的错误信息。（不利于debug）

------

*What we're considering is wether or not we should change this. Which is a step in our never ending quest to find the right balance between "fix and cleanup old things" and "do not break old projects". In this case we're wondering what you think. For Unity5 we have been working on the ability for Unity to automatically update your scripts (more on this in a subsequent blogpost). Unfortunately, we would be unable to automatically upgrade your scripts for this case. (because we cannot distinguish between "this is an old script that actually wants the old behaviour", and "this is a new script that actually wants the new behaviour").*

我们正在考虑是改变这一点，我们一直在“修复和清理旧代码”和“不要破坏老代码”之间平衡。（想改但不好改）

------

*We're leaning towards "remove the custom == operator", but are hesitant, because it would change the meaning of all the null checks your projects currently do. And for cases where the object is not "really null" but a destroyed object, a nullcheck used to return true, and will if we change this it will return false. If you wanted to check if your variable was pointing to a destroyed object, you'd need to change the code to check "if (myObject.destroyed) {}" instead. We're a bit nervous about that, as if you haven't read this blogpost, and most likely if you have, it's very easy to not realise this changed behaviour, especially since most people do not realise that this custom null check exists at all.**[3]***

我们倾向于移除自定义的==运算符，但是犹豫，因为这可能会改变你当前项目内所有判空检查的含义。比如一个并不是真的null但是在C++层已经被销毁的对象，做空检查会返回true，改了的话会返回false，如果你想检查变量是否指向一个被销毁的对象，你需要改用"if (myObject.destroyed){}"。关于这一点我们有点紧张。因为如果你没读过这篇博客，甚至可能即使读过了，也很容易忽略这种行为的改变，尤其是因为大多数人根本没有意识到这种自定义空检查的存在。

------

[1] We do this in the editor only. This is why when you call GetComponent() to query for a component that doesn't exist, that you see a C# memory allocation happening, because we are generating this custom warning string inside the newly allocated fake null object. This memory allocation does not happen in built games. This is a very good example why if you are profiling your game, you should always profile the actual standalone player or mobile player, and not profile the editor, since we do a lot of extra security / safety / usage checks in the editor to make your life easier, at the expense of some performance. When profiling for performance and memory allocations, never profile the editor, always profile the built game.

我们只在编辑器模式中这样做。这就是为什么当调 GetComponent()查询一个不存在的Component时，你会看到发生C#内存分配，因为我们在新分配的"fake null object"中生成了自定义警告字符串。这种内存分配不会在构建的游戏中发生（应该指已经出包的版本）。这是一个很好的例子，说明了为什么如果你要对游戏进行性能分析，你应该始终对实际的独立播放模式或移动的播放模式下进行性能分析（？），而不是在编辑器模式下进行性能分析，因为我们在编辑器模式中做了很多额外的安全/保护/使用检查，让你用起来更方柏霓，但代价是一些性能损耗。当进行性能和内存分配分析时，不要对编辑器进行分析，而应该对构建的游戏进行分析。

------

[2] This is true not only for GameObject, but everything that derives from UnityEngine.Object

这发生在所有派生自UnityEngine.Object的对象上

------

[3] Fun story: I ran into this while optimising GetComponent<T>() performance, and while implementing some caching for the transform component I wasn't seeing any performance benefits. Then [@jonasechterhoff](https://twitter.com/jonasechterhoff) looked at the problem, and came to the same conclusion. The caching code looks like this:

```c#
private Transform m_CachedTransform
public Transform transform
{
  get
  {
    if (m_CachedTransform == null)
      m_CachedTransform = InternalGetTransform();
    return m_CachedTransform;
  }
}
```

Turns out two of our own engineers missed that the null check was more expensive than expected, and was the cause of not seeing any speed benefit from the caching. This led to the "well if even we missed it, how many of our users will miss it?", which results in this blogpost :)

有趣的故事：作者在给GetComponent<T>()方法做优化时，在给Transform做缓存时并没有看到预想的缓存优化，错误代码如上所示。也是这篇博客诞生的原因。

------

自己试了下(Unity2019)

```c#
var temp = new GameObject("test_object");
Object.DestroyImmediate( temp );
Debug.Log(temp == null);//true
```

````c#
var temp = new GameObject("test_object");
Object.Destroy( temp );
Debug.Log(temp == null);//这里返回false，可能因为调用的并不是DestroyImmediate，temp在打印前被GC掉了
````

看来Unity始终是没有解决这个问题啊。。。更别说Object.destroyed了
