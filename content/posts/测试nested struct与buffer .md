---
title: "Test behavior of nested struct in Unity ECS"
date: 2019-06-19T14:40:12+08:00
toc: false
images:
tags:
- Unity
- ECS
- Unit Test



---



Official ForEach test -> Unity.Entities.Tests/ForEach/ForEachGeneralTests.cs

### Nested Struct Test

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



### UpdateBufferVar Test

```C#
using NUnit.Framework;

[TestFixture]
public class UpdateBufferVarTest : ECSTestsFixture
{
    [Test]
    public void _0_Update_One_Var()
    {
        var entity = m_Manager.CreateEntity(typeof(U));

        World.GetOrCreateSystem<UpdateBufferWithMultipleElement>().Update();
        var target = m_Manager.GetBuffer<U>(entity)[0].Var1;
        Assert.AreEqual(1, target);
    }

    [Test]
    public void _1_Update_Second_Var()
    {
        var entity = m_Manager.CreateEntity(typeof(U));

        World.GetOrCreateSystem<UpdateBufferWithMultipleElement>().Update();
        var target = m_Manager.GetBuffer<U>(entity)[0].Var2;
        Assert.AreEqual(2,target);
    }

    [Test]
    public void _2_Modify_Existing_Element()
    {
        var entity = m_Manager.CreateEntity(typeof(U));
        World.GetOrCreateSystem<ModifyExistingElement>().Update();
        var target = m_Manager.GetBuffer<U>(entity)[0].Var2;
        Assert.AreEqual(2,target);
    }
}
```



### System

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
/// <summary>
/// How to Iterate Buffer? Can buffer contain more than one variables?
/// </summary>
public class UpdateBufferWithMultipleElement : ComponentSystem
{
    protected override void OnUpdate()
    {
        Entities.ForEach ((DynamicBuffer<U> u)=>
        {
            var element = new U { Var1 = 1,Var2 = 2};
            u.Add(element);
        });
    }
}

/// <summary>
/// How to modify existing element?
/// </summary>
public class ModifyExistingElement : ComponentSystem
{
    protected override void OnUpdate()
    {
        Entities.ForEach((DynamicBuffer<U> u) =>
        {
            var element = new U {Var2 = 1};
            u.Add(element);
            var temp = u[0];
            temp.Var2 += 1;
            u[0] = temp;
            
            // THIS DOESN'T WORK
            // u[0].Var1 = 2;
        });
    }
}

```

​    

### Conclusion

1.  Nested struct can be simply modify without further operation.

2.  Buffer element can contains more than one variables.

3.  Buffer iteration in ComponentSystem is different from Component iteration

```C#
Entities.ForEach((ref Foo foo)=>{}); // iterate component

Entities.ForEach((DynamicBuffer<Fooo> fooo)=>{}); // iterate buffer
```

4.  Buffer has no syntax sugar as nested struct, so I should modify it carefully.