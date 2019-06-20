---
title: "如何使用Unity ECS打造Reactive System？"
date: 2019-06-19T14:40:12+08:00
toc: true
images:
tags:
- Unity
- ECS
- Tutorial
- Reactive System
- System Design


---



## 结构变化

所谓结构变化，即是Entity添加删除Component，诸如此类。追踪办法有两种。

### 1. 使用SystemStateComponent

#### 原理

所谓State，含义是只能被手动删除的Component，在删除Entity后，依然留下做一些殿后工作，只有被指名要求删除的时候才会被删除。这种特性就让我们可以通过不同的Query去获得结构变化的消息。

#### 栗子

1.  假设我们有一个Entity，身上有两个组件，A：IComponentData 与 B: ISystemStateComponentData
2.  当我们添加A组件的时候，通过Filter{ 有A无B }，我们可以在别处得知这个Entity何时被添加。在添加后手动加入B组件。
3.  当我们删除Entity，或者移除A组件的时候，通过Filter { 有B无A }，同理可得知何时这个Entity被移除或是A组件被移除。

更具体的实现可以在查看官方对于ParentSystem的设计。

### 2. 查询ComponentVersion

每当出现某Component相关的结构性变化的时候，该Component的版本就会+1 。

```C# 
EntityManager.GetComponentVersion()
```

## 数据变化

顾名思义。方法有三种。

### 1. Chunk检查

#### 原理

1.  GlobalSystemVersion为记录一个世界所有系统更新信息的版本号。在每一个系统更新**之前**，GSV++。
2.  LastSystemVersion为系统记录自己的版本号。在某系统更新**之后**，它会保存GSV，含义是**上次**运行时的版本号，直到下次某系统更新之后，它的版本号不会更改
3.  每一种Component，在System申请写入权限的时候，都会记录该System的LSV
    获取方式为chunk.GetArch 

因此，if ( ComponentVer > LSV ) 说明该Component被修改了（有系统获得了写入权限）。而之所以不用考虑等于，是因为除非只有1个系统，不然是不会有等于的情况的，不能理解的话可以画图思考一下。

注意这个信息时效性只有一帧，从上次该系统更新后到这次更新后的一帧，因此在这次更新中，修改Component后查询是否改变，答案是True，反之为False。

#### 举例

1.  系统的更新顺序为A->B->C->A
2.  那么GSV ：0 -> 1 -> 2 -> 3，每个系统更新之前+1 
3.  当数据在B系统被写入，Component就记住了B的GSV = 1
4.  当我们在第二次轮到A系统的时候监测是否Component被改动，DidChange自动使用A系统上次的GSV记录 LSV = 0 与 Component记录的信息CV = 1做对比，发现CV > LSV，得知信息已经被更改了，返回True。

#### API

```c#
chunk.DidChange(InputAType, LastSystemVersion);
```

注意LSV应从EntityManager.LastSystemVersion取得，并传入Job

```C#
[BurstCompile]
struct UpdateJob : IJobChunk
{
   public ArchetypeChunkComponentType<InputA> InputAType;
   public ArchetypeChunkComponentType<InputB> InputBType;
   [ReadOnly] public ArchetypeChunkComponentType<Output> OutputType;
   public uint LastSystemVersion;

   public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
   {
       var inputAChanged = chunk.DidChange(InputAType, LastSystemVersion);
       var inputBChanged = chunk.DidChange(InputBType, LastSystemVersion);
       if (!(inputAChanged || inputBChanged))
           return;
      //...
}
```



####  2. Query自动检查

在声明Query的时候，特别注明

```C#
m_Group.SetFilterChanged(new ComponentType{ typeof(InputA), typeof(InputB)});
```



这样Query就会把没被修改的ComponentType排除在外。注意，这种检查是Component层级，而不是单个Entity层级的。

```c#
EntityQuery m_Group;
protected override void OnCreate()
{
   m_Group = GetEntityQuery(typeof(Output), 
                               ComponentType.ReadOnly<InputA>(), 
                               ComponentType.ReadOnly<InputB>());
   m_Group.SetFilterChanged(new ComponentType{ typeof(InputA), typeof(InputB)});
}
```



### 3. IJobForEach中使用 [ChangeFilter]

与Query的排除效果类似。

#### 示例

```c#
public struct ProcessTendency : IJobForEachWithEntity<HumanState, HumanStock>
{
    public void Execute(Entity entity, int index, [ChangedFilter] ref State state)
    {
```





## Reference

[UNITY - ECS Packages](https://docs.unity3d.com/Packages/com.unity.entities@0.0/manual/chunk_iteration_job.html)

[designing-an-efficient-system-with-version-numbers/ - 5argon](https://gametorrahod.com/designing-an-efficient-system-with-version-numbers/)











