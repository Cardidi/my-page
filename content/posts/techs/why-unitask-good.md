---
title: "为什么我们需要UniTask？谈论异步与Unity。"
date: 2022-04-09T11:30:03+00:00
description: "因为这玩意是真的好用……"
# weight: 1
# aliases: ["/first"]
tags: ["Unity", "UniTask", "C#", "客户端开发", "异步"]
categories: "技术"
#showToc: true
#TocOpen: false
draft: false
#hidemeta: false
#comments: false
#canonicalURL: "https://canonical.url/to/page"
#disableHLJS: false
#hideSummary: false
#searchHidden: false
#ShowPostNavLinks: true
#cover:
#    image: "<image path/url>" # image path/url
#    alt: "<alt text>" # alt text
#    caption: "<text>" # display caption under cover
#    relative: false # when using page bundles set this to true
#    hidden: true # only hide on current single page
#editPost:
#    URL: "https://github.com/Cardidi/my-page/content"
#    Text: "在 Github 上查看原Markdown"
#    appendFilePath: true
---

> 本文原本是在TDS写的一篇分享文章，现在我拿到我的博客上给大家分享一下。

我之前曾经提到过一个Unity插件叫做UniTask，是用来提供Unity异步操作用的。但是我想各位可能不太明确的一个问题是，为什么Unity需要使用到异步操作？或者还有人会问我，什么是异步？为了讲清楚这个东西，我将会从C#的异步和Unity多线程上的陷阱开始，为大家一步一步梳理UniTask应用场景。
多线程与异步

大家都知道线程这么一回事：程序运行的时候需要运行在至少一个线程上，才能保证程序中所定义的算法可以被执行。

而且稍微有经验的人也知道，操作多线程是一件比较麻烦的事情。一方面是线程安全问题——两个线程同时操作同一处内存时会引发不确定的操作结果——另一方面是如果一个线程需要依赖另外一个线程的结果，那我们还不停判断另外一个线程的状态来指导当前线程执行下一步算法的时机。

第一个问题相对而言会好解决一些：通过锁，保证多个线程操作同一块内存时，操作必定是原子的。但是第二个问题就会麻烦很多，其主要表现在以下方面：


- 状态机有可能会写错，增加debug成本。
- 会导致单个线程不断去判断另外一个线程的状态，使之空转浪费宝贵的CPU资源。
- 如果另外一个线程迟迟不能完成，这将会阻塞这个线程，有可能导致整个程序被卡死。
- 会让我们写的代码不够直观，增加编码成本（回调地狱）。

因此，为了解决上面的几个问题，我们提出了一种叫做异步的策略。

# 异步是什么

我觉得网上讲异步讲的都不够直接，我就直接的抛出我所认为异步的本质：异步是用来转让程序运行权的工具。我下面举几个具体的代码例子来示意：

```csharp
void Block()
{
    ... // 做一件非常耗时间的事情
    Console.WriteLine("Done");
}

// 什么是async
async Task BlockAsync()
{   
    await Task.Run(() => Block()); // 为什么这里有await
    return null;
}

void CallThem()
{
    Block(); // 这个会导致执行这一个函数的线程被阻塞
    var task = BlockAsync(); // 而这个却不会
    Console.WriteLine("Called");
}

```

当我调用```CallThem()```这个函数时你会发现```Block()```会把这个程序卡死，导致你要等很久才能看到第一个“Done”被打印出来；但是你如果删掉```CallThem()```内的```Block()```调用，这个程序却在一瞬间就把“Called”给打印到控制台，不过要等很久控制台上面才会打印“Done”。这是为什么呢？

Magic在于这条一行

```csharp
await Task.Run(() => Block());
```

在这行，await代表的是“我要转让运行权给调用我的函数”，也就是说将程序的运行权从函数BlockAsync转让回调用它的CallThem，从而让CallThem可以继续执行下去。不过你也需要注意一点，如果一个函数里面会出现await，那么其函数头必须使用async去标记，具体的缘由后面再说。

顺带一提，Task在这里指的是一个任务。需要注意的是只有可以返回```Task```、```ValueTask```或者```TaskAwaiter```的方法才可以使用```await```去转让运行权。同时，我们通过```Task.Run()```将我们的计算放置在了另外一个线程中，当被```await```的```Task```执行完成后，这个线程将会接着执行这个函数剩下的部分，直到它遇到了```return```或者```await```。

因此不难看出，通过```await```这一个魔法就能保证我们最大限度地不让程序被卡住，同时还能让我们尽可能榨干一个线程——干不完你我就先不鸟，你好了我再来鸟你。

## 我想要返回一个值

不过上面那句话还不太准确，因为我们还有可能需要这个函数所返回的值——说不准是我们求他干完而不是想不鸟就不鸟它——比如这样子：

```csharp
int Calc()
{
    ... // 又要做一件非常耗时间的事情
    return result; // result = 114514
}

async Task<int> CalcAsync()
{
    return Task.Run(() => Calc());
}

async Task CallAgain()
{
    var result = await CalcAsync(); // 这里的result可是一个int哦，神奇不？
    Console.WriteLine($"Result is {result}");
}
```

这个时候我们去调用```CallAgain()```时，你会发现一个神奇的事情——为什么我要等很久，控制台上面才会有“Result is 114514”出现？因为我在这里

```csharp
var result = await CalcAsync();
```

使用了```await```，导致运行权转移到调用```CallAgain()```的函数手上。当你```await```的函数执行完成并返回结果时，由于```await```的是一个```Task<int>```，并且是一个右值，因此编译器会帮你把返回的```int```放置到左值内，从此让你获得114514这一个整数并打印在屏幕中。

从这两个例子，我想你应该明白了为什么我要说异步本质上是在讨论程序的运行权问题。

## 一个特别的语法糖

在你继续看下去之前，我还需要补充一个语法糖——下面这两种写法是等价的：

```csharp
int Calc() {...} // 一个折磨人的计算

Task<int> ResultA() // 不包含async的函数头
{
    return Task.Run(() => Calc());
}

async Task<int> ResultB() // 包含了async的函数头
{
    int result = await Task.Run(() => Calc()); // 我故意单独写成一行，方便大家理解
    return result;
}
```

你会注意到ResultA和ResultB的函数头都要求返回了一个```Task<int>```，但实际上ResultB返回的是一个```int```。这里是一个语法糖，如果一个函数标记为```async```的话，并且其返回值类型为```Task<T>```或者类似的表达，那么这个函数可以返回T类型对象。原因在于编译器在这里会自动的帮我们生成一个```Task<T>```对象，并且在这个函数第一次使用```await```时将会给调用者返回```Task<T>```对象。

因此，```async```实际上是一种对于函数的语法糖，没有```async```一样可以实现异步，但是在语法上会有少许变化。

总之，你只需要记牢这一个本质：异步是用来转让程序运行权的工具。

# Unity的多线程陷阱

刚刚我们聊完了异步，这个时候你应该能够注意到异步这个策略可以有效地缓解多线程编程中的一些痛点。但我也多次强调，异步是用来转让程序运行权的工具。我并没有强调异步是专门为多线程而设计的，我仅仅是强调他在转让运行权上的特点，而接下来，我将会以这一个特点来分析为什么Unity的多线程存在陷阱。

## Task.Run()陷阱

如果你写一段这样子的代码

```csharp
Task AddRB2(params GameObject[] objs)
{
    return Task.Run(() =>
    {
        foreach (var o in objs)
            o.AddComponent<Rigidbody2D>();
    });
}
```

那你一定会收到来自Unity的毒打（前提是必须await函数所返回的Task，否则异常将不被捕获）：

```
UnityException: Internal_AddComponentWithType can only be called from the main thread.
Constructors and field initializers will be executed from the loading thread when loading a scene.
Don't use this function in the constructor or field initializers, instead move initialization code to the Awake or Start function.
```

Unity的报错里面有一些很有意思的地方：要求UnityEngine API必须从主线程调用。而这里的主线程就是指驱动MonoBehaviour的生命周期函数的线程。我之前提到过一点，使用```Task.Run()```会把要计算的内容放置到另外一个线程内进行计算，因此这个问题也不难理解，但是你想在其他线程内调用UnityEngine API的梦想就要破碎了。

不过事实真的如此吗？

## Unity协程

事实上，Unity倒是巧妙地利用了协程（也就是你在MonoBehaviour里面调用的XXXCoroutine()这一类函数）实现了你所希望的“类”多线程。他的本质上是在每一帧的某个时间点轮询是否存在需要在此执行的Coroutine，从而将我们的计算分布到不同帧或者时机内。但是从本质上，由于你依旧是在同一个线程上执行代码，这个方法最多看起来像，实际上却是另外一回事。

因此很遗憾，你确实没有比较好的办法在其他线程调用UnityEngine API。

而且你也知道，协程写起来实际上非常让人不爽。问题如下：

- 写法相对而言较复杂，不直观。
- 如果需要对状态进行查询，需要自己设置标记位，不简洁。
- 无法脱离MonoBehaviour存在，不方便。

还记得异步吗？异步似乎刚刚好能够解决前两个问题：写起来简洁明确，返回的Task自带状态查询，那这岂不是我们理想中的最佳方案吗？

## UniTask的应用

UniTask是目前解决这个问题的最佳方案，它在保证Unity线程安全的前提下还支持C#的异步编程模型，而且不需要依赖MonoBehaviour便可以运行。
不过我想偷个小懒，这个是用法的文档： https://github.com/Cysharp/UniTask#readme

顺便在这里放一段我是用UniTask的代码片段（非常的赏心悦目，清晰易读）：

```csharp
// 这一个函数是场景切换页面所用的代码，调用这个代码可以切换页面并提供切换的淡入淡出动画。
public async Task Transit()
{
    group.alpha = 0;
    await UniTask.Yield(); // 我这里等待下一帧的Update再正式开始执行动画

    try
    {
        await group.DOFade(1, duration.x).ToUniTask(); // 这里是Dotween的动画API，还支持UniTask
        var asset = await sceneTask; // 这里是等待场景的加载
        
        if (beforeOpenSceneAct != null) await beforeOpenSceneAct();
        asset.OpenScene(); // 在这里正式切换场景
        if (afterOpenSceneAct != null) await afterOpenSceneAct();
    }
    catch (Exception e)
    {
        Debug.LogError(e);
        return;
    }

    await group.DOFade(0, duration.y).ToUniTask(); // 然后淡出动画
    gameObject.SetActive(false); // 我不担心线程安全，因为我实际上还是在主线程中哦！
}

public async void Start()
{
    await Transit(); // 这里可不会阻塞主线程
}
```

我们常规的```Task.Run()```会导致我们的代码跑在子线程上，但是为什么UniTask可以保证线程安全呢？这个时候我就可以回收这个伏笔：异步是用来转让程序运行权的工具。

我可没说过异步是专门用在多线程上面的。如果有JavaScript网页开发基础，并且知道Promise模型，那么此处是一点即通：UniTask实际上是可以使用主线程或者子线程来计算```await```之后的代码。

我刚刚展示的所有代码实际上都是在主线程中执行的，因此这不会导致线程安全问题。除此之外，它还有以下特点：

- 性能极佳，官方宣称Zero Allocation；
- 在观感上，你会觉得这样子写会非常的赏心悦目——没有到处yield return的困扰。
- 通过对UniTask.Yield()进行await，可以获取到远比MonoBehaviour更加细致的生命周期（见 https://github.com/Cysharp/UniTask#playerloop ）。
- 无需MonoBehaviour的StartCoroutine()来启动一个协程。
- 可以对UniTask.SwitchToXXX()进行await，来切换你想运行的线程（比如主线程或者线程池）。
- 对UnityEngine API做了UniTask的包装，可以通过非常简洁的方式对UnityEngine原生API进行await。
- 对第三方插件的支持也挺不错，比如我们刚刚看到的DoTween。

换言之，UniTask可以非常完美的结合Unity协程和C# Task，通过优雅的方式来提升我们的工作效率和功能实现。我个人愿意称其为最佳方案！
