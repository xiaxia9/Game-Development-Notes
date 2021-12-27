# 1 前言

在了解动画系统之前，笔者有很多问题，比如：动画是如何动起来？我们看到的角色的位置是如何更新的？动画是如何更新的？等等。在看源码的时候，又遇到新的问题，比如：Skeleton和Mesh之间的关系是什么？。所以决定看看源码~

接下来，介绍下本文的结构，本文一共分为四节，分别是：

- 基本概念：列举了一些结构及解释。
- 流程讲解：本节结合源码分析了组件注册、动画更新和渲染流程。不过，不包括Sequence等类型的动画资源、蒙太奇同步、蒙太奇触发等的具体分析。
- 问题及回答：由于一些问题不方便在源码流程中讲解，所以单独出了一节。比如：Mesh坐标在何时何处设置的？Skeleton与Mesh之间的关系等等~
- 总图：这一节是本篇文章的精华部分，也映照了本文的标题。本节将动画系统流程以图绘的方式展示，使用函数节点表示，可以更方便的定位到函数并查看。老地方，图片及pdf版在github上哦：https://github.com/xiaxia9/blogs/tree/master/UE4

注：使用UE4.26，且本文仅讨论骨骼动画。

# 2 基本概念

下面的内容部分参考于：https://docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/Rendering/Overview/

|               结构                | 解释                                                         |
| :-------------------------------: | ------------------------------------------------------------ |
|           USkeletalMesh           | 绑定到可以为使网格变形而设置动画的骨骼层级骨架的几何体。 由两部分组成：构造网格表面的多边形和可用于为多边形设置动画的层级骨架。 |
|      FSkeletalMeshRenderData      | 渲染数据，包括LOD渲染数据等。                                |
|    FSkeletalMeshLODRenderData     | 骨骼蒙皮LOD渲染数据，包括蒙皮权重、皮肤权重配置文件数据等。  |
|          USceneComponent          | 需要添加到 FScene 中的任意对象的基础类，如光照、网格体、雾等。 |
|        UPrimitiveComponent        | 基元组件，是可视性和相关性确定的基本单位。可渲染或进行物理交互（碰撞检测）的任意资源的基础类。也可以作为可视性剔除的粒度和渲染属性规范（投射阴影等）。与所有 UObjects 一样，游戏线程拥有所有变量和状态，渲染线程不应直接对其进行访问。 是一些可见几何体的父类，比如：  ShapeComponents (Capsule, Sphere, Box)：用于碰撞检测但不渲染。 StaticMeshComponent、SkeletalMeshComponent：包含渲染的预构建几何体，也可用于碰撞检测。 |
|          UMeshComponent           | 继承自UPrimitiveComponent，是所有具有可渲染的三角形集合实例的组件的抽象基类，如：UStaticMeshComponent、USkeletalMeshComponent。 |
|       UStaticMeshComponent        | 继承自UMeshComponent，用于创建UStaticMesh实例，静态网格是由一组静态多边形组成的几何体。 |
|       USkinnedMeshComponent       | 继承自UMeshComponent，支持骨骼蒙皮网格渲染，不支持动画。提供USkeletalMesh接口等。 |
|      USkeletalMeshComponent       | 继承自USkinnedMeshComponent，创建带动画的SkeletalMesh资源的实例。 支持动画。 |
|              FScene               | UWorld 的渲染器版本。对象仅在其被添加到 FScene（注册组件时调用）后才会存在于渲染器中。渲染线程拥有 FScene 中的所有状态，游戏线程无法直接对其进行修改。 主要操作是添加或删除图元（Primitive）和灯光。 |
|       FPrimitiveSceneProxy        | UPrimitiveComponent 的渲染器版本，为渲染线程映射 UPrimitiveComponent 状态。存在于引擎模块中，用于划分为子类以支持不同类型的基元（骨架、刚体、BSP 等）。实现某些非常重要的函数，如 GetViewRelevance、DrawDynamicElements 等。 |
|        FPrimitiveSceneInfo        | 内部渲染器状态（FRendererModule 实现专有），对应于 UPrimitiveComponent 和 FPrimitiveSceneProxy。存在于渲染器模块中，因此引擎看不到它。单个UPrimitiveComponent的内部渲染器状态。 |
| FDynamicSkelMeshObjectDataGPUSkin | 存储顶点蒙皮所需的更新矩阵。                                 |
|    FSkeletalMeshObjectCPUSkin     | CPU蒙皮网格渲染数据。                                        |
|     FSkeletalMeshObjectStatic     | 静态网格渲染数据。                                           |
|    FSkeletalMeshObjectGPUSkin     | GPU蒙皮渲染数据。包含FDynamicSkelMeshObjectDataGPUSkin* DynamicData，它用于动态更新和渲染。 |
|     FGPUBaseSkinVertexFactory     | 存储皮肤处理所需的骨基质，以及对 GPU 皮肤顶点工厂着色器代码需要用作输入的各种顶点缓冲区的引用。 |
# 3 流程讲解

先梳理一下大致的流程，流程可以分为三个部分，分别是组件注册、动画更新和渲染。

组件注册：在Game线程中执行。在游戏启动时，需要先注册USkeletalMeshComponent组件，然后调用执行注册事件。注册事件做了以下内容：初始化Anim、更新Animation、刷新骨骼变换、创建渲染状态并分配MeshObject、创建物理状态。

动画更新：判断是否需要执行蒙太奇更新优化；进行预更新——重置动画通知队列和RootMotionBlendQueue，赋值；蒙太奇更新、蒙太奇同步组更新、蓝图更新。分发并行执行和并行执行结束两个任务。在并行执行Task中，更新动画蓝图中的节点和TickAssetPlayer()；在并行执行结束Task中，更新数据，以便之后在创建渲染命令时可以判断。

渲染：游戏线程创建渲染命令，渲染线程执行渲染命令，最后更新骨骼数据。

下面是详细介绍：

## 3.1 组件注册

在介绍具体流程前，首先放一下官方文档https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/Rendering/ThreadedRendering/中的两段话，了解了之后再介绍流程：

- **UPrimitiveComponent** 是属于可被渲染、可投射阴影资源的基础游戏线程类，拥有其自身的可视状态等属性。渲染线程无法直接触及 UPrimitiveComponent 的内存，因为游戏线程可能随时写入其构件中。渲染线程自身拥有代表此功能的类 - **FPrimitiveSceneProxy**。游戏线程被创建和注册后，无法触及 FPrimitiveSceneProxy 的内存构件。**UActorComponent::RegisterComponent** 将一个组件添加到场景，并创建一个 FPrimitiveSceneProxy 使其对渲染器可见。组件注册后，如其为可见，将为所需的每次通路调用 **FPrimitiveSceneProxy::DrawDynamicElements**。
- 动态资源更新的一个最佳范例是游戏线程动画每帧生成的骨骼网格体骨骼变形。目的是：在每个动画更新进渲染线程上（在此可将变形设为着色器常数）的一个阵列后，从游戏线程中获取变形。如在每帧更新索引或顶点缓冲，结果相同。以下是操作顺序：
  - **USkinnedMeshComponent::CreateRenderState_Concurrent** 分配 **USkinnedMeshComponent::MeshObject**。从此时起，游戏线程只可写入 **MeshObject** 指示器，但不可写入 **FSkeletalMeshObject** 的内存。
  - **USkinnedMeshComponent::UpdateTransform** 每帧被调用至少一次，更新组件的移动。在 GPU 蒙皮中将调用 **FSkeletalMeshObjectGPUSkin::Update**。现在游戏线程上拥有最新的变形，需要将它们转移到渲染线程中。操作方法：首先在堆（**FDynamicSkelMeshObjectData**）上分配内存，然后将骨骼变形复制进去，再使用 ENQUEUE_UNIQUE_RENDER_COMMAND_TWOPARAMETER 将此拷贝传到渲染线程。渲染线程现在拥有此拷贝，并负责删除。ENQUEUE_UNIQUE_RENDER_COMMAND_TWOPARAMETER 宏包含复制变形到最终目的地的代码，因此变形可被设为着色器常数。如更新顶点位置，这就是锁定和更新顶点缓冲的位置。
  - 在一些情况下，组件会被分离。游戏线程使渲染命令入列，以释放所有动态 FRenderResources，现在可将 MeshObject 指示器设为 NULL；然而实际内存仍被渲染线程引用，无法删除。此时延迟删除机制即可发挥作用。从 **FDeferredCleanupInterface** 派生的类可按对线程无害的异步方式进行删除。FSkeletalMeshObject 应用此接口。游戏线程需要开始 FSkeletalMeshObject 的延迟删除，因此它调用了 **BeginCleanup(MeshObject)**。安全且完成清理后，内存将被逐步删除。

下面是具体流程：

1、首先是序列化FSkeletalMeshRenderData。从UObject::ConditionalPostLoad()开始，最后调用到FSkeletalMeshRenderData::Serialize()进行序列化。

2、然后是游戏启动，从UActorComponent::RegisterComponent()开始，主要是将组件添加到场景中，并创建一个FPrimitiveSceneProxy使其对渲染器可见。流程如下：

- 调用ExecuteRegisterEvents()执行注册事件；
- 调用ComputeRequiredBones()使用RenderData（渲染数据）、RequiredVirtualBones（虚拟骨骼）、PhysAssetBones（物理骨骼）填充OutRequiredBones，之后清除不可见的Bones，然后添加Mirror需要的Bones、SocketBones、ShadowShapeBones；
- 调用TickAnimation()主动更新一次动画；
- 然后调用RefreshBoneTransforms()刷新Bone位置；
- 调用CreateRenderState_Concurrent()创建渲染状态，并分配MeshObject：FSkeletalMeshObjectStatic、FSkeletalMeshObjectCPUSkin or FSkeletalMeshObjectGPUSkin；其解释是负责将骨骼变换、变形目标状态等发送到渲染线程的对象。这一步和渲染建立关系。
- 调用UpdateBounds()更新边界；
- 调用CreateSceneProxy()创建一个FPrimitiveSceneProxy使其对渲染器可见；
- 调用FSkeletalMeshObjectGPUSkin::Update()进行GPU渲染更新；
- 调用UpdatePreviousRefToLocalMatrices()初始化FDynamicSkelMeshObjectDataGPUSkin结构。

## 3.2 动画更新

### 3.2.1 TickComponent

1、从USkeletalMeshComponent::TickComponent()中开始，调用USkinnedMeshComponent::TickComponent()，再调用USkeletalMeshComponent::TickPose()，最后在USkeletalMeshComponent::TickAnimation()中对LinkedInstances（链接的动画蓝图）、AnimScriptInstance（动画蓝图）、PostProcessAnimInstance（从SkeletalMesh的PostPhysicsBlueprint属性创建的实例）三种类型的动画实例执行UpdateAnimation()更新。

![](pictures\3-TickAnimation.png)

2、动画更新做了哪些内容？UpdateAnimation

- 在UpdateAnimation()函数中，当我们设置了OnlyTickMontagesWhenNotRendered且未被渲染时，会调用UpdateMontage()更新，然后返回。这一步其实是在进行优化。

- 如果不是，则往下执行。先调用**PreUpdateAnimation()**进行预更新，在预更新函数里，会重置动画通知队列和RootMotionBlendQueue，并调用FAnimInstanceProxy::PreUpdate()进行代理更新。PreUpdate()其实就是进行常规赋值操作，比如给RootMotionMode、SkelMeshCompLocalToWorld、UngroupedActivePlayers、SyncGroups、ComponentTransform、ComponentRelativeTransform、ActorTransform等赋值。

  ![](pictures\3-PreUpdateAnimation.png)

- 之后来到UpdateMontage()，这里主要是调用Montage_UpdateWeight()根据DeltaTime进行动画权重更新和调用Montage_Advance()处理一些RootMotion逻辑——提取位移（具体可以参照5 总图或上篇关于RootMotion的文章）。

  ![](pictures\3-UpdateMontage.png)

- 然后调用UpdateMontageSyncGroup()更新蒙太奇同步组，计算同步组Leader。

- 然后调用UpdateMontageEvaluationData()更新蒙太奇data，重置计算数据，然后将之前计算的权重等数据放在Proxy中。之后会用到。

- 然后调用BlueprintUpdateAnimation()进行蓝图更新。

- 如果动画需要立即更新，还需要调用ParallelUpdateAnimation()和PostUpdateAnimation()进行并行更新，这些在后面会详细介绍。

3、之后回到上层USkinnedMeshComponent::TickComponent()中继续执行，如果ShouldUpdateTransform(bLODHasChanged) 则执行RefreshBoneTransforms()刷新骨骼坐标，收集动画数据。这里主要做了以下三件事：

- 给FAnimationEvaluationContext AnimEvaluationContext赋值。

- 预并行处理：调用PreEvaluateAnimation()(和上面的一样，对三种类型进行操作：AnimScriptInstance、LinkedInstance、PostProcessAnimInstance），这个函数是给FAnimInstanceProxy结构中的MainInstanceProxy、Skeleton赋值。

- 分发并行执行任务：

  - 如果bDoParallelEvaluation并行执行为true，则调用DispatchParallelEvaluationTasks()，在这个函数里面调用SwapEvaluationContextBuffers()将我们在上面操作的AnimEvaluationContext与本地缓存进行交换，然后创建FParallelAnimationEvaluationTask执行任务和FParallelAnimationCompletionTask完成任务。

  ![](pictures\3-SwapContext.png)

  - 反之，调用DoParallelEvaluationTasks_OnGameThread()，正如函数名所示，这里在Game线程中主动调用。这里先调用SwapEvaluationContextBuffers()交换，然后调用ParallelAnimationEvaluation()，最后再调用SwapEvaluationContextBuffers()交换回来。最后回到上层调用PostAnimEvaluation()。

![](pictures\3-ParallelGameThread.png)

4、之后回到上层USkinnedMeshComponent::TickComponent()中，如果我们设置了AlwaysTickPose，则调用DispatchParallelTickPose()。

### 3.2.2 Task

在上面的DispatchParallelEvaluationTasks()中，我们创建了FParallelAnimationEvaluationTask和FParallelAnimationCompletionTask两个Task，这两个Task也都是在任务线程中执行的。

1、首先先看FParallelAnimationEvaluationTask。

- 调用DoTask()，一路执行到USkeletalMeshComponent::PerformAnimationProcessing，该函数首先调用**UAnimInstance::ParallelUpdateAnimation()**进行动画代理更新和Tick，分别是下面两个函数：

  - UpdateAnimation()：动画蓝图节点更新。从根节点依次遍历更新。

  - TickAssetPlayerInstances()：主要执行TickAssetPlayer()来更新FAnimAssetTickContext，RootMotion在此提取并更新。我们也可以自己实现TickAssetPlayer()来适配项目需求。

- 然后调用EvaluateAnimation()：执行动画蓝图更新。

- 之后调用EvaluatePostProcessMeshInstance()：更新PostProcessMeshInstance。

- 之后调用FinalizePoseEvaluationResult()：填充OutBoneSpaceTransforms和OutRootBoneTranslation。

- 最后调用FillComponentSpaceTransforms()：给RootBone赋值，将InBoneSpaceTransforms转换为OutComponentSpaceTransforms

2、最后是FParallelAnimationCompletionTask。调用DoTask()，执行到CompleteParallelAnimationEvaluation()，先调用SwapEvaluationContextBuffers()交换回来，然后调用PostAnimEvaluation()，最后发送给渲染线程。

## 3.3 渲染

在上面TickAnimation之后，渲染线程需要渲染动画数据，以便我们可以看到角色。

在FSkeletalMeshObjectGPUSkin::Update()中我们可以看到有ENQUEUE_RENDER_COMMAND这样一个宏，这个宏是处理两个线程间通讯的问题。它的作用是创建一个渲染命令，游戏线程将该命令插入渲染命令队列，渲染线程在开始时调用执行函数。

游戏线程：调用DoDeferredRenderUpdates_Concurrent()，然后调用SendRenderTransform_Concurrent()，判断动画更新流程中FParallelAnimationCompletionTask任务标记的bRenderDynamicDataDirty是否为True，如果为Ture，则需要将组件变换发送到渲染器。然后判断是否需要Update()，最后调用FSkeletalMeshObjectGPUSkin::Update()创建渲染命令，以便渲染线程执行。

渲染线程：从FRenderingThread::Run()开始，之后调用FSkeletalMeshObjectGPUSkin::Update()更新FDynamicSkelMeshObjectDataGPUSkin数据并执行渲染命令，最后调用FGPUBaseSkinVertexFactory::FShaderDataType::UpdateBoneData()更新骨骼数据。

# 4 问题及回答

## 4.1 角色动画的类型

下面的内容来自于《游戏引擎架构》11.1 角色动画的类型

游戏引擎中有3种最常用的动画技术，下面将分开介绍：

- 赛璐璐动画：赛璐璐是透明的塑料片，可在上面绘画。把一连串含动画的赛璐璐放置于固定的手绘背景之上，就能产生动感，而无须不断重复绘制静态的背景。赛璐璐动画的电子版本是称为精灵动画的技术。所谓精灵，其实是一张小位图，叠在全屏的背景影响之上而不会扰乱背景，通常由专门的图形硬件绘制。精灵是二维游戏时代最主要的技术。

- 刚性层阶式动画：角色由一堆刚性部分建模而成，这些刚性部分以层阶形式彼此约束，类似于哺乳类动物以关节连接骨骼，这样能使角色自然地移动。但是，这种技术的最大问题在于，角色的身体会在关节位置产生碍眼的裂缝。

  ![](pictures\4-刚性层阶动画.png)

- 蒙皮动画：是当今最流行的技术，也是本篇文章介绍的重点。在蒙皮动画中，骨骼（skeleton）是由刚性的骨头（bone）所建构而成的，这与刚性层阶动画是一样的。然而，这些刚性的部件并不渲染显示，始终是隐藏起来的。称为皮肤（skin）的圆滑三角形网格会绑定于骨骼上，其顶点会追踪关节（joint）的移动。蒙皮上每个顶点可按权重绑定至多个关节，因此当关节移动时，蒙皮可以自然地拉伸。

## 4.2 胶囊体、Skeleton（骨骼、骨架）、Bone（骨头、joint）、Mesh（皮）之间的关系

在了解动画系统之前，我们需要了解下一些用语之间的关系~

1、对于移动来说，无论是Code Driven还是Data Driven，都是先调用StartNewPhysics()修改USceneComponent* UpdatedComponent的位置，然后在FScopedMovementUpdate析构时更新Mesh（作为子组件更新）的位置。这个UpdatedComponent一般默认是根组件，也就是我们所说的胶囊体。我们在游戏中是无法直观的看到胶囊体的，可以使用show Collision查看。如下所示：

![](pictures\4-Collision.png)

2、而Skeleton是由Bone构成的一种层级结构。如下面所示的USkeleton资源，我们选择一个Skeleton中的一个Bone，图中显示的是pelvis，在右方Details框中可以看到坐标。（所以位移信息是存储在Skeleton里面的哦，不是Mesh~）

![](pictures\4-Skeleton.png)

3、接下来是Mesh，Mesh仅仅是一层皮而已，我们所说的修改Mesh坐标其实是修改的根骨骼的位置和朝向，因为坐标等信息是保存在骨骼里的。当然，RootMotion动画中的位移也是写在骨骼中的。Mesh是依附于骨骼存在的。如下面所示的USkeletonMesh资源，可以看到编辑器右上角显示的标签是Mesh。

![](pictures\4-Mesh.png)

## 4.3 SkeletalMeshComponent坐标是在何时何处被设置的？

在游戏中，我们看到的玩家其实是Mesh，那它的坐标是在哪设置的呐？

1、首先介绍一下层级关系，如下图所示（角色蓝图），可以看到组件之间的层级关系，Mesh是CapsuleComponent的子组件，而CapsuleComponent也是根组件。

![](pictures\4-MeshComponent层次结构.png)

2、其次，在移动组件中的PerformMovement()函数里面可以看到FScopedMovementUpdate这样一个结构，在这个结构体析构的时候，会调用EndScopedMovementUpdate()，然后继续执行；当位置有改变时，调用PropagateTransformUpdate()，最后一路调用到UpdateComponentToWorldWithParent()更新坐标，这里是直接设置的坐标。如下图所示：

![](pictures\4-MeshComponent坐标更新.png)

3、**所以默认来说，主控端SkeletalMeshComponent的坐标是相对于父组件的，其坐标的变换随着父组件相对变化。**从移动角度来说，移动组件Tick时，执行PerformMovement()，然后就会判断是否需要更新SkeletalMeshComponent，也就是说SkeletalMeshComponent的坐标每帧都在跟随着父组件变化。在之前的移动同步流程中，我们发现，模拟端是不执行PerformMovement()的，也没调用到FScopedMovementUpdate结构，所以它重写了一套Mesh插值平滑流程。

## 4.4 USkeletalMesh资源与USkeleton资源介绍

在第一个问题中，解释了Skeleton和Mesh之间的关系。之后，我们发现UE4提供了两种资源，分别是USkeletalMesh和USkeleton，现在介绍一下它们~

1、USkeleton：维护了一个骨骼内部的层级关系及Bone信息。主要成员如下：

- TArray< struct FBoneNode> BoneTree成员：维护骨骼层级结构。存储名字和父节点id。

- FReferenceSkeleton ReferenceSkeleton成员：这个结构记录了Bone Info（名字和父节点id）、Bone Transform、Virtual Bone Data等，是操作的基础。也印证了之前所说的坐标信息保存在骨骼里这句话。
- TArray< FVirtualBone> VirtualBones成员：虚拟骨骼的基本信息，记录SourceBoneName、TargetBoneName和VirtualBoneName。

2、USkeletalMesh：绑定到骨架后的网格体。由两部分组成，构造网格表面的多边形和可用于为多边形设置动画的层级骨架。主要成员如下：

- TSharedPtr< FSkeletalMeshModel> ImportedModel：模型的几何信息

- **TUniquePtr< FSkeletalMeshRenderData> SkeletalMeshRenderData成员：渲染数据。**
- USkeleton* Skeleton成员：同上，所属骨架。
- TArray< FSkeletalMaterial> Materials：材质信息
- TArray< struct FSkeletalMeshLODInfo> LODInfo：LOD信息
- class UPhysicsAsset* PhysicsAsset：物理资源设置
- TArray< FTransform> RetargetBasePose：重定向pose的缓冲区
- TArray<UClothingAssetBase*> MeshClothingAssets：布料资源配置
- TArray< FSkinWeightProfileInfo> SkinWeightProfiles：mesh的蒙皮权重配置

## 4.5 USkinnedMeshComponent介绍

USkinnedMeshComponent是USkeletalMeshComponent的父组件，也是我们要深入了解的骨骼蒙皮技术的核心内容。继承于UMeshComponent，可渲染。主要成员如下：

- **class USkeletalMesh* SkeletalMesh：Skeleton Mesh资源**
- TArray< ESkinCacheUsage> SkinCacheUsage：皮肤缓存功能
- TArray< FVertexOffsetUsage> VertexOffsetUsage：顶点偏移功能

USkeletalMeshComponent继承于USkinnedMeshComponent，用于创建SkeletalMesh资源的实例。支持动画，并多了如下的主要成员：

- UAnimInstance* AnimScriptInstance：动画实例
- FVector RootBoneTranslation：根骨骼位置
- FBlendedHeapCurve AnimCurves：Curve临时存储
- TArray<UAnimInstance*> LinkedInstances：关联的动画实例
- TArray< FTransform> CachedBoneSpaceTransforms：BoneSpace下的缓存
- TArray < FTransform> CachedComponentSpaceTransforms：ComponentSpace下的缓存
- uint16 CachedAnimCurveUidVersion：Curve缓存，用于识别是否需要更新

## 4.6 Mesh Tick选项优化

在角色蓝图中，可以对Mesh组件设置Visibility Based Anim Tick Option，可选的选项分别是：

- AlwaysTickPoseAndRefreshBones：无论是否渲染，总是tick和Refresh Bone transforms。
- AlwaysTickPose：总是tick，只在渲染时Refresh Bone transforms。
- OnlyTickMontagesWhenNotRendered：只在渲染时tick pose和Refresh Bone transforms。 不渲染时，只更新montage，跳过其他操作。
- OnlyTickPoseWhenRendered：只在渲染时tick和Refresh Bone transforms。

![](pictures\4-MeshTick.png)

（动画优化时，应该会用到这部分的内容叭~

## 4.7 动画中的位移提取流程

首先需要动画师写入位移。

以RootBone为例。

在ParallelUpdateAnimation()并行执行的过程中，调用了FAnimInstanceProxy::TickAssetPlayerInstances()，如果是UAnimSequence会最终调用到ExtractRootMotion()进行提取，UAnimMontage会调用ExtractRootMotionFromTrackRange()提取，UAnimCompositeBase会调用ExtractRootMotionFromTrack()提取。

以UAnimSequence为例，会调用到TickAssetPlayer()设置Time；之后调用ExtractRootMotionFromRange()来提取一段范围内的StartTransform和EndTransform；然后会调用GetBoneTransform()，这个函数里会根据FCompressedAnimSequence.CompressedDataStructure（压缩骨骼数据流）等信息生成FAnimSequenceDecompressionContext（封装骨骼压缩编解码器使用的解压相关数据）DecompContext，DecompContext会把Time作为参数传递给Seek()，在这里为RelativePos赋值，最后调用DecompressBone()提取FTransform，该函数里面调用的GetBoneAtomTranslation()、GetBoneAtomRotation()均用到了RelativePos。所以是使用动画中的Time作为参数，然后进行提取。

## 4.8 蒙皮介绍

下面的内容部分来自于《游戏引擎架构》11.5 蒙皮及生成矩阵调色板

蒙皮：把三维网格顶点联系至骨骼的过程。蒙皮用的网格是通过其顶点系上骨骼的。每个顶点可绑定至一个或多个关节。若某顶点只绑定至一个关节，它就会完全跟随该关节移动。若绑定至多个关节，则该顶点的位置就等于把它逐一绑定至个别关节后的位置，再取其加权平均。

要把网格蒙皮至骨骼，三维建模师必须替每个顶点提供以下额外信息：

- 该顶点要绑定到的（一个或多个）关节索引。
- 对于每个绑定的关节，提供一个权重因子，以表示该关节对最终顶点位置的影响力。

如同计算其他加权平均时的习惯，每个顶点的权重因子之和为1。

蒙皮矩阵：能把网格顶点从原来位置（绑定姿势）变换至骨骼的当前姿势。

书上重点的内容介绍完了，那么上面的概念又对应于虚幻引擎中的哪部分内容呐？

这里先介绍几个结构体及概念：

- FSkinWeightDataVertexBuffer：存储骨骼索引/权重数据的顶点缓冲区。该结构继承于FVertexBuffer，FVertexBuffer又继承于FRenderResource，也就是说这是渲染线程拥有的渲染资源。
- FSkinWeightLookupVertexBuffer：存储皮肤权重流偏移/影响计数的查找顶点缓冲区。和上面的FSkinWeightDataVertexBuffer结构一样，也是渲染资源。

- FSkinWeightVertexBuffer：皮肤权重数据顶点缓冲区和查找顶点缓冲区的容器。也就是说，包含FSkinWeightDataVertexBuffer和FSkinWeightLookupVertexBuffer成员。
- FSkinWeightProfileInfo：存储面向用户的属性的结构，用于识别SkeletalMesh级别的配置文件。
- FSkeletalMeshLODRenderData：骨骼蒙皮LOD渲染数据，主要包括下面内容：
  - FSkinWeightVertexBuffer SkinWeightVertexBuffer：蒙皮权重
  - FSkinWeightProfilesData SkinWeightProfilesData：皮肤权重配置文件数据
  - TArray< FBoneIndexType> ActiveBoneIndices：活动骨骼

FSkeletalMeshRenderData里有FSkeletalMeshLODRenderData成员，FSkeletalMeshLODRenderData里有FSkinWeightVertexBuffer记录了蒙皮权重。在编辑器启动时，首先从USkeletalMesh::PostLoad()开始，然后调用FSkeletalMeshRenderData::Serialize()，然后调用FSkeletalMeshLODRenderData::Serialize()进行序列化，之后再调用SerializeStreamedData()，然后执行Ar << SkinWeightVertexBuffer，再执行Ar << VertexBuffer.DataVertexBuffer，最后调用VertexBuffer.WeightData->Serialize(Ar)完成FSkinWeightDataVertexBuffer中WeightData和Data的赋值；之后回到上层PostLoad()，然后调用FSkeletalMeshRenderData::InitResources()，最后调用FSkeletalMeshLODRenderData::InitResources()进行初始化。

# 5 总图

![](pictures\【UE4】图解动画系统源码.png)

# 6 参考

《游戏引擎架构》第11章  动画系统

[图形编程](https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/Rendering/)

[Skinned Mesh原理解析和一个最简单的实现示例](https://happyfire.blog.csdn.net/article/details/3105872)

[UE4动画系统更新源码分析](https://zhuanlan.zhihu.com/p/405437842)

[UE4 动画系统 源码及原理剖析](https://blog.csdn.net/qq_23030843/article/details/109103433)



动画系统的学习之路仅仅开了一个头，还有很多东西需要去学习，本文也有很多细节没有涉及到，以后慢慢学习慢慢补充~

水平有限，如有错误，烦请指正，谢谢吼~

