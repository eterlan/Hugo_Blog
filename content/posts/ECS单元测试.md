---
title: "ECS 单元测试教学"
date: 2019-06-19T14:40:12+08:00
toc: false
images:
tags:
- Unity
- ECS
- Unit Test


---

网上的教学视频有些过时，因此想写一个新的教学，教大家如何对自己的ECS代码进行单元测试。

### 第一步 打开Unity单元测试的界面

位于Window/General/TestRunner。

### 第二步 建立TestFolder

位于Assets/Create/Test/Test Assembly Folder

FAQ：

1.  为什么要用asmdef？

我们只希望测试这一小片代码，因此不希望每次测试都让unity重新编译其他部分，因此只添加几个测试需要的依赖文件，这样可以让测试跑的很快。

2.  为什么要勾上Test Assemblies？

这样可以在打包时忽略这块代码，范围是该文件夹以内的代码。

3.  为什么只勾上Editor？

因为我们只在编辑器内需要这块代码。

4.  写好待测试的系统system与组件component

这里待测试的是ref关键字能否作用到嵌套struct中的内容。 

#### system

```C#
using Unity.Entities;

public class UpdateNestedStruct : ComponentSystem
{
    protected override void OnUpdate()
    {
        Entities.ForEach((ref T t) =>
        {
            t.point.X += 1;
            t.forTest += 1;
        });
    }
}
```

#### component

```C#
using System;
using Unity.Entities;

[Serializable]
public struct T : IComponentData
{
    public Point point;
    public int forTest;
}

public struct Point
{
    public int X;
    public int Y;
}
```



### 写单元测试

#### 准备工作

翻一翻源码，可以看到Unity自个儿的单元测试是继承自ECSTestFixture，其目的是做一些准备工作。我们也有样学样，否则的话World 和 EntityManager都是空值。

一翻尝试之后，发现无法直接继承或者是调用Unity.Entities.Test这个库。解决办法是把ECSTestsFixture和TestComponent从Unity.Entities.Test中复制到自己的测试文件夹。

#### 手动Update自己的系统

```C#
using NUnit.Framework;

[TestFixture]
public class UpdateNestedStructTests : ECSTestsFixture
{
    [Test]
    public void _0_Update_Normal_Var()
    {
        var entity = m_Manager.CreateEntity(typeof(T));

        World.GetOrCreateSystem<UpdateNestedStruct>().Update();
        var target = m_Manager.GetComponentData<T>(entity).forTest;
        Assert.AreEqual(1,target);
    }

    [Test]
    public void _1_Update_Nested_Struct()
    {
        var entity = m_Manager.CreateEntity(typeof(T));
        
        World.GetOrCreateSystem<UpdateNestedStruct>().Update();
        var target = m_Manager.GetComponentData<T>(entity).point.X;
        Assert.AreEqual(1,target);
    }
}
```

写好测试，我们回到编辑器，选中需要测试的代码，并点击Run Selected。

这样我们就写好一个ECS专用的单元测试了！

可以看到，手动更新系统的方式，让ECS与笨重的MonoBehaviour的测试方式区别很大。这让我们可以随意地测试每小块代码。

### reference

-   5argon
    <https://medium.com/@5argon/unity-ecs-unit-testing-problems-with-ecs-8f31c7a37386> 
-   infallible
    <https://www.youtube.com/watch?v=Ibj7O_fQXKs> 
-   Discuss
    https://forum.unity.com/threads/how-to-unit-test-where-is-the-unity-entities-tests-namespace.540251/