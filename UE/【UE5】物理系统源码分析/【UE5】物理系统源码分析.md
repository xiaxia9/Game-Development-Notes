# 1. 前言

和之前的文章一样，主要还是以SkeletalMesh为例。本文包括物理初始化、物理更新、物理销毁、碰撞检测系统、以SkeletalmeshComponent例子分析物理场景与游戏场景交互、Chaos、碰撞检测等流程。

补充：

- 物理相关的部分概念及刚体物理初始化流程可以先参考我的另一篇文章[【UE5】基于物理的角色动画（基础篇）](https://zhuanlan.zhihu.com/p/568049696)。
- 如果有想了解动画系统的朋友，可以参考我的另一篇文章[【UE4】图解动画系统源码](https://zhuanlan.zhihu.com/p/446851284)。
- 本文篇幅较长，是因为介绍了大量的流程代码，内容是从零开始介绍，然后逐渐深入一丢丢，结合代码阅读本文更佳。

# 2. 流程介绍

## 2.1 物理初始化

**创建物理场景：**首先在世界初始化时，调用UWorld::CreatePhysicsScene()创建物理场景FPhysScene_Chaos。物理场景提供了一些功能接口，包括碰撞事件通知、物理同步、添加力等，接口实现会调用chaos底层，其实也就是游戏场景和物理场景之间的一个交互，具体交互调用流程会在后面分析。

![创建物理场景流程](pictures\创建物理场景.png)

**创建物理状态：**然后在组件注册时，调用USkeletalMeshComponent::OnCreatePhysicsState()创建物理状态。在该函数中，创建物理状态分为两种方案：

- 一种方案是不启用逐三角面碰撞（EnablePerPolyCollision），这也是引擎默认的方案。在该方案下，会使用物理资产中的数据进行物理初始化。首先创建物理资产中的USkeletalBodySetup数量的Aggregate，之后使用物理资产中的SkeletalBodySetups和ConstraintSetup数据给SkeletalMeshComponent的**TArray<struct FBodyInstance* > Bodies**和**TArray<struct FConstraintInstance*> Constraints** 进行赋值，最后初始化物理资产内部的实体之间的物理碰撞关系。
- 另外一种方案则是启用逐三角面碰撞。在该情况下，使用SkeletalMesh的BodySetup数据进行初始化，此方案开销较大。

![创建物理状态流程](pictures\创建物理状态.png)

向物理引擎中的FPBDRigidsSolver中注册初始化刚体：

![](pictures\RegisterObject.png)

**加入物理场景：**我们需要将对象传递给物理线程才可以进行模拟，该流程在创建物理状态时调用。有两种函数接口用于加入物理场景，分别是AddObject()和AddToComponentMaps()，在AddToComponentMaps()中做了物理代理和组件的一个映射。IPhysicsProxyBase类为物理代理基类，所有物理代理需要继承于该类。ComponentToPhysicsProxyMap存储组件到物理代理的映射，有的Actor组件可能会存在多组物理代理，比如SkeletalMeshComponent。PhysicsProxyToComponentMap存储物理代理到组件的映射。流程如下图所示：

- 加入物理场景接口：FSkeletalMeshPhysicsProxy等继承于TPhysicsProxy， TPhysicsProxy继承于IPhysicsProxyBase。

  ![](pictures\加入物理场景接口.png)

- StaticMesh：

  ![](pictures\StaticMeshAddToPhysicsScene.png)

- SkeletalMesh：

  ![SkeletalMesh加入物理场景](pictures\SkeletalMeshAddToPhysicsScene.png)

## 2.2 物理更新

UE5中有TickGroup概念，TickGroup指定在引擎的一帧中，该Tick在该帧中的执行顺序。具体解释请见官网：https://docs.unrealengine.com/5.0/zh-CN/actor-ticking-in-unreal-engine/。

TickGroup可以分为以下几类：

- TG_PrePhysics：在物理模拟开始前执行。
- TG_StartPhysics：物理模拟开始执行。
- TG_DuringPhysics：物理模拟并行执行。
- TG_EndPhysics：物理模拟结束执行。
- TG_PostPhysics：在刚体和布料模拟结束后执行。
- TG_PostUpdateWork：帧内所有更新完成之后执行。
- TG_LastDemotable：被延迟到最后执行。
- TG_NewlySpawned：所有Tick组执行完毕后，对所有世界中新生成的物体Tick

流程如下所示：

![](pictures\WorldTick.png)

具体代码流程为，在每次**UWorld::Tick()**时：

- **UWorld::SetupPhysicsTickFunctions()**
  - 设置StartPhysicsTickFunction和EndPhysicsTickFunction的bCanEverTick参数为true，启动Tick。

  - 如果未注册，则调用RegisterTickFunction()将StartPhysicsTickFunction注册到TG_StartPhysics组，将EndPhysicsTickFunction注册到TG_EndPhysics组。
  - 调用DeferredPhysResourceCleanup()执行物理引擎资源清理，目前仅对PHYSICS_INTERFACE_PHYSX处理。
  - 调用FChaosScene::SetUpForFrame()设置重力和FPhysicsSolver的MMaxDeltaTime、MMaxSubSteps（物理子步）、MMinDeltaTime。
- **FTickTaskManager::StartFrame()**
  - 设置FTickContext Context。

  - 调用FTickTaskSequencer::StartFrame()重置数据，包括CleanupTasks、TickCompletionEvents、TickTasks、HiPriTickTasks、WaitForTickGroup。

  - 调用FTickTaskManager::FillLevelList()设置TArray<FTickTaskLevel*> LevelList。流程为遍历传入的参数Levels，将有效的Level添加至LevelList中。
  - 遍历LevelList，调用FTickTaskLevel::StartFrame()。
    - 设置FTickContext Context。
    - 调用FTickTaskLevel::ScheduleTickFunctionCooldowns()。根据Cooldown对TickFunctionsToReschedule进行升序排序，然后遍历TickFunctionsToReschedule，设置TickState为CoolingDown，然后添加至AllCoolingDownTickFunctions列表中，最后重置TickFunctionsToReschedule。
    - 遍历AllCoolingDownTickFunctions，确定此帧哪些CoolingDownTickFunction可以执行。
  - 遍历LevelList，调用FTickTaskLevel::QueueAllTicks()。
    - 遍历AllEnabledTickFunctions。
      - 调用FTickFunction::QueueTickFunction()。
        - 在该函数中遍历该TickFunction的前置条件TArray< struct FTickPrerequisite> Prerequisites，递归调用FTickFunction::QueueTickFunction()。然后使用FTickFunction调用GetCompletionHandle()获取FGraphEventRef对象，然后添加至前置任务TaskPrerequisites中。
        - 调用FTickTaskSequencer::QueueTickTask()。在该函数中调用StartTickTask()创建Task，然后调用AddTickTaskCompletion()将该Task添加到HiPriTickTasks或TickTasks数组中，将FGraphEventRef对象添加到TickCompletionEvents数组中。
      - 如果调用TickInterval>0， 则调用RescheduleForInterval()重新设置时间间隔。
    - 遍历AllCoolingDownTickFunctions。如果TickState为Enabled，则调用FTickFunction::QueueTickFunction()和FTickTaskLevel::RescheduleForInterval()，细节同上。
- **UWorld::RunTickGroup(TG_PrePhysics)**
  - 调用UWorld::RunTickGroup()
    - 调用FTickTaskManager::RunTickGroup()。
      - 调用FTickTaskSequencer::ReleaseTickGroup()释放该TickGroup。在该函数中，调用FTickTaskSequencer::DispatchTickGroup()，遍历HiPriTickTasks\[WorldTickGroup][IndexInner]和TickTasks\[WorldTickGroup][IndexInner]，并调用TGraphTask< FTickFunctionTask>::Unlock()，然后重置数组。回到上一级函数，阻塞等待，如果TickCompletionEvents[Block]数量大于0，调用WaitUntilTasksComplete()直到任务完成，然后调用ResetTickGroup()重置数据
      - 如果bBlockTillComplete为True，查询是否有新生成的Tick，如果无，则设置bFinished为true。
    - TickGroup更新。
  - 上面提到了将**StartPhysicsTickFunction**注册到TG_PrePhysics组，这里介绍一下该TickFunction的流程：入口为FStartPhysicsTickFunction::ExecuteTick()，然后调用UWorld::StartPhysicsSim()，再调用FChaosScene::StartFrame()发起物理模拟
    - 调用FPhysScene_Chaos::OnStartFrame()。
      - 调用FPhysicsReplication::Tick()。根据复制目标，更新body的状态。
      - 调用FPhysScene_Chaos::UpdateKinematicsOnDeferredSkelMeshes()。收集需要移动的Actor和Bodies的transform，并批量处理。
      - 广播OnPhysScenePreTick。
      - 广播OnPhysSceneStep。
    - 调用FChaosSolversModule::GetSolversMutable()，获取SolverList。如果GetSolver()存在，则将该值添加到SolverList中。
    - 遍历SolverList，调用Chaos::FPhysicsSolverBase::AdvanceAndDispatch_External()获取FGraphEventRef，然后将FGraphEventRef添加到CompletionEvents中。在AdvanceAndDispatch_External()函数中会调用SetSolverSubstep_External()设置物理子步，在后面会介绍。


- **UWorld::RunTickGroup(TG_StartPhysics)**
  - 同上。
- **UWorld::RunTickGroup(TG_DuringPhysics, false)**
  - 不阻塞等待，其余同上。
- **UWorld::RunTickGroup(TG_EndPhysics)**
  - 同上。
  - 上面提到了将**EndPhysicsTickFunction**注册到TG_EndPhysics组，这里介绍一下该TickFunction的流程：入口为FEndPhysicsTickFunction::ExecuteTick()
    - 调用GetPhysicsScene()获取物理场景FPhysScene。
    - 调用GetCompletionEvents()获取物理完成事件FGraphEventArray。
    - 调用IsCompletionEventComplete()判断事件是否完成。如果未完成，则调用DontCompleteUntil()延迟事件。如果已完成，则调用FinishPhysicsSim()，最后调用FChaosScene::EndFrame()结束该帧的执行。
- **UWorld::RunTickGroup(TG_PostPhysics)**
  - 同上。

- **FTickTaskManager::EndFrame()**
  - 调用FTickTaskSequencer::EndFrame()，是否打印日志。
  - 设置bTickNewlySpawned为false。
  - 遍历LevelList数组，调用FTickTaskLevel::EndFrame()。在该函数中，调用FTickTaskLevel::ScheduleTickFunctionCooldowns()，细节同上。
  - 重置LevelList。

除了上面的介绍之外，还有很多的TickFunction，我们可以直接在代码中全局搜索来看到各TickGroup有哪些操作；也可以在代码执行时，通过查看FTickTaskLevel的AllEnabledTickFunctions、AllCoolingDownTickFunctions、AllDisabledTickFunctions、NewlySpawnedTickFunctions参数来获取。

![](pictures\TickGroupShow.png)

AActor通过控制成员变量PrimaryActorTick来控制Tick的执行时序，这里总结一下各TickGroup常见的模块执行：

- **TG_PrePhysics：**APawn、AActor、APlayerController、UMovementComponent、URadialForceComponent、USkeletalMeshComponent、USkeletalMeshComponent::ClothTickFunction、UNiagaraComponent、FSkinWeightProfileManager、USkinnedMeshComponent、UCharacterMovementComponent::PrePhysicsTickFunction、AGameMode、UTimelineComponent
- **TG_StartPhysics：**UWorld::StartPhysicsTickFunction
- **TG_DuringPhysics：**ALandmassActor、UUIFrameworkPlayerComponent、UNiagaraComponent、UActorComponent、ULensComponent、USceneCaptureComponentCube、UParticleSystemComponent、FParticleSystemWorldManager、AHUD、UGameplayTasksComponent、ALandscapeBlueprintBrushBase
- **TG_EndPhysics：**UWorld::EndPhysicsTickFunction、USkeletalMeshComponent::EndPhysicsTickFunction
- **TG_PostPhysics：**UPointCloudComponent、UDisplayClusterSyncTickComponent、UCharacterMovementComponent::PostPhysicsTickFunction、USkeletalMeshComponent的EndTickGroup、USpringArmComponent、UParticleSystemComponent的EndTickGroup、UPhysicsSpringComponent、UChaosEventListenerComponent、AChaosSolverActor、
- **TG_PostUpdateWork：**AControlRigControlActor、AGameplayAbilityTargetActor_Trace、FSkinWeightProfileManager、UChaosDebugDrawComponent

继承TickFunction的结构体如下所示：

![](pictures\TickFunction.png)



## 2.3 物理结束

如果开启了物理动画，且SkeletalMeshComponent结束物理模拟后，会调用FSkeletalMeshComponentEndPhysicsTickFunction::ExecuteTick() -> USkeletalMeshComponent::EndPhysicsTickComponent()，在该函数中调用BlendInPhysicsInternal()进行物理Pose混合或调用SyncComponentToRBPhysics()使用物理Pose更新骨骼Pose。

当游戏结束运行时，会调用UWorld::FinishDestroy() -> UWorld::ReleasePhysicsScene()来释放World拥有的PhysicsScene。销毁Actor时会调用DestroyPhysicsState()。

![](pictures\DestroyPhysicsState.png)

## 2.4 物理子步

物理子步的开启及其他设置请参考官方文档https://docs.unrealengine.com/5.0/zh-CN/physics-sub-stepping-in-unreal-engine/。UE5的物理子步的流程也和UE4有一些不同。

由于UWorld::StartPhysicsSim()是由UWorld::Tick()调用的，所以物理更新也受World的DeltaTime影响。物理子步可以把一帧拆成多个SubStep来更新，这样提升了物理模拟的精确度，但也损耗了性能。

在UWorld::Tick() -> UWorld::SetupPhysicsTickFunctions() -> FChaosScene::SetUpForFrame()时进行物理子步参数的计算：

- 计算物理子步模拟的总时间：MDeltaTime = FMath::Min(InDeltaSeconds, InMaxSubsteps * InMaxSubstepDeltaTime);
- 调用SetMaxDeltaTime_External(InMaxSubstepDeltaTime)设置一次SubStep的最大时间。
- 调用SetMaxSubSteps_External(InMaxSubsteps)设置物理子步的最大次数。
- 调用SetMinDeltaTime_External(InMinPhysicsDeltaTime)设置最小物理时间。

在UWorld::StartPhysicsSim() -> FChaosScene::StartFrame() -> Chaos::FPhysicsSolverBase::AdvanceAndDispatch_External()中分发SubStep任务：

- 调用SetSolverSubstep_External()设置是否启用物理子步。
- 如果NumSteps大于0，则调用FPBDRigidsSolver::PushPhysicsState()函数设置物理状态。在该函数中，调用MarshallingManager.GetProducerData_External()获取FPushPhysicsData* ProducerData数据，进行部分设置，然后调用FChaosMarshallingManager::Step_External()继续设置ProducerData数据，之后在Step_External()函数中将ProducerData添加到TArray<FPushPhysicsData*> ExternalQueue数组中。一个SubStep对应一个ProducerData数据。
- 在While循环中，不断调用MarshallingManager.StepInternalTime_External()获取我们在上一步设置的ProducerData。然后调用TGraphTask< FPhysicsSolverProcessPushDataTask>::CreateTask()创建任务，并添加到Prereqs中。之后调用TGraphTask< FPhysicsSolverAdvanceTask>::CreateTask()创建任务。

上面已经为每一个SubStep创建完任务，接下来便是任务的调度，后面会详细介绍：

- FPhysicsSolverProcessPushDataTask::ProcessPushData()
- FPhysicsSolverAdvanceTask::AdvanceSolver()

## 2.5 物理交互

其实前面也能总结出来物理场景与游戏场景交互，这里用USkeletalMeshComponent例子帮助理解。流程如下：

- 物理模拟前。将几何数据发送到PhysicsScene。在上面的物理初始化流程中，已经知道USkeletalMeshComponent从调用OnCreatePhysicsState()，到调用FPhysScene_Chaos::AddToComponentMaps()加入物理场景。

- 物理模拟开始。动画更新后，将骨骼数据发送给物理场景。调用MarkForPreSimKinematicUpdate()标记更新，DeferredKinematicUpdateSkelMeshes存储组件数据，流程如下：

  ![](pictures\StartSim.png)

- 物理模拟中。在StartPhysicsSim()时，会调用UpdateKinematicsOnDeferredSkelMeshes()进行更新，在该函数中：（PS：这一块的内容较多，会在后面详细介绍......）

  - 遍历DeferredKinematicUpdateSkelMeshes，如果没有开启逐三角面碰撞，将每一个有效的Body的IPhysicsProxyBase存储在ProxiesToDirty数组中；如果开启了，则调用UpdateKinematicBonesToAnim()直接更新。
  - 当ProxiesToDirty数组的数量大于0时，获取ProxiesToDirty[0]的解算器，然后将代理数组作为参数传递给解算器。
  - 获取SkelComp的数据，设置Chaos::FRigidBodyHandle_External。当我们添加一个力的时候，FBodyInstance::AddForce() -> FPhysScene_Chaos::AddForce_AssumesLocked() -> TThreadedSingleParticlePhysicsProxyBase(FRigidBodyHandle_External的父类)::AddForce()。 也就是说，该Handle也提供了一些接口用于响应游戏场景的调用，直到最后影响PBDRigidParticles。该Handle是游戏场景和物理场景数据交互的重要角色。

  - 调用UpdateActorsInAccelerationStructure()更新加速结构。

  - 重置DeferredKinematicUpdateSkelMeshes。

  ![](pictures\DuringSim.png)

- 物理模拟结束。会触发FSkeletalMeshComponentEndPhysicsTickFunction执行物理Pose与骨骼Pose的处理。以调用的BlendInPhysicsInternal()为例，当bParallelBlend为true时候，调用SwapEvaluationContextBuffers()交换AnimEvaluationContext和组件缓存的动画数据，FAnimationEvaluationContext即缓存数据，然后创建FParallelBlendPhysicsTask、FParallelBlendPhysicsCompletionTask任务。

  - FParallelBlendPhysicsTask::DoTask() -> ParallelBlendPhysics() -> PerformBlendPhysicsBones()。更新Pose。在PerformBlendPhysicsBones()函数中，遍历InRequiredBones，调用GetUnrealWorldTransform_AssumesLocked()获取物理姿势PhysTM（该函数最终调用获取的是Handle的数据），当物理权重大于0时，然后调用UpdateWorldBoneTM()递归更新WorldBoneTMs而获取ParentWorldTM，计算PhysTM相对父骨骼的Transform，然后Blend Transform，并存放到AnimEvaluationContext中。

    ![](pictures\FParallelBlendPhysicsTask.png)

  - FParallelBlendPhysicsCompletionTask::DoTask() -> CompleteParallelBlendPhysics()。SwapEvaluationContextBuffers()会交换回AnimEvaluationContext和缓存的动画数据，AnimEvaluationContext现在存储的是更新后的组件缓存的动画数据。调用FinalizeAnimationUpdate()完成动画更新，推送给渲染线程。

    ![](pictures\FParallelBlendPhysicsCompletionTask.png)

- 还有，当角色移动时，调用OnMovementUpdated()时，最后会调用SendPhysicsTransform()将组件的世界变换发送给物理场景。流程如下：

![](pictures\ActorMove.png)

# 3. Chaos模块

ps：本节不会提及过多的理论知识，相关理论知识会标注来源，可自行选择查看。仅介绍Chaos底层的一些重要的数据结构及接口。

## 3.1 核心数据

前面的章节阅读完之后，已经了解了游戏场景和物理场景的上层交互流程。

Chaos模拟是基于位置的，PBD的理论知识请阅读[【深入浅出 Nvidia FleX】(1) Position Based Dynamics](https://zhuanlan.zhihu.com/p/48737753)。

通过上文，可以知道游戏世界和物理引擎的交互通过Task实现了多线程。

在Chaos引擎中，刚体是使用粒子来描述的。以刚体为例，其继承顺序为：

- TParticles：粒子父类。

- TGeometryParticlesImp：包含几何参数的粒子。

- TKinematicGeometryParticlesImp：包含运动参数的粒子。

- TRigidParticles：刚体。

和粒子相对应的还有Handle，通过Handle来获取粒子，从而对粒子的数据进行处理。以刚体为例，其继承顺序为：

- TParticleHandleBase
- TGeometryParticleHandleImp
- TKinematicGeometryParticleHandleImp
- TPBDRigidParticleHandleImp

在Handle里还可以看到Proxy成员，Proxy连接着游戏世界和物理世界。以刚体为例，其继承顺序为：

- IPhysicsProxyBase
- FSingleParticlePhysicsProxy：（using FPhysicsActorHandle = Chaos::FSingleParticlePhysicsProxy*;）
- TThreadedSingleParticlePhysicsProxyBase
- FRigidBodyHandle_External，FRigidBodyHandle_Internal

还有很重要的一个概念是解算器。FPBDRigidsSolver继承于FPhysicsSolverBase，是物理计算的核心，从命名也可以看出来是基于PBD的。

在前面的物理子步章节中提到了物理的Task流程，我们提到了FPhysicsSolverProcessPushDataTask::ProcessPushData()，这个Task会传递FPushPhysicsData参数。FPushPhysicsData保存着需要更新到物理场景的数据以及一些Callback。

![](pictures\PushPhysicsData.png)

我们还提到了FPhysicsSolverAdvanceTask::AdvanceSolver()，在该函数中，会调用到Chaos的核心函数AdvanceOneTimeStepImpl()。

![](pictures\CoreFunction.png)

FChaosMarshallingManager：从成员变量就可以看出，它管理着FPushPhysicsData。

## 3.2 碰撞检测系统

游戏引擎的碰撞系统紧密地和物理引擎整合。相关理论知识可查看《游戏引擎架构》12.3节，本节不会提及过多的理论知识。

从AdvanceOneTimeStepImpl()来看：

- Chaos::FPBDRigidsEvolutionGBF::Integrate()：对FPBDRigidsSOAs这块内存中的粒子进行操作，在每次迭代时更新刚体的位置、形状、加速度、AABB等。

![update AABB](pictures\UpdateAABB.png)

- 加速结构类型：

  - AABBTree：

    ![](pictures\AABBTree.png)

  - BoundingVolume：

    ![](pictures\BoundingVolume.png)

  

  - BoundingVolumeHierarchy：

    ![](pictures\BVH.png)

  - SpatialAccelerationCollection：

    ![](pictures\SpatialAccelerationCollection.png)

- 碰撞检测阶段：

  - 粗略阶段（broad phase）：使用轴对齐包围盒（AABB）粗略判断哪些物体有机会碰撞。

    ![](pictures\BroadPhase.png)

  - 中间阶段（midphase）：检测复合形状的逼近包围体。在下一级FSingleShapePairCollisionDetector::GenerateCollisionImpl()函数中，会调用Collisions::UpdateConstraint()更新narrow phase的约束。生成碰撞的流程如下所示：

    ![](pictures\MidPhase.png)

  - 精确阶段（narrow phase）：通过精确计算接触点，穿透距离等检测两个物体是否真的相交，并创建出对应的碰撞约束。

    - 在GeometryQueries.h文件中可以看到两种查询方式，分别是OverlapQuery和SweepQuery。以UCharacterMovementComponent::ComputeFloorDist()为例，然后调用UWorld::SweepSingleByChannel()，然后调用物理层接口，最后调用到GJKRaycast2()执行检测。
  - 其他没看....
  
- 创建约束图：描述粒子之间的关系。Edge为约束，Node为粒子。遍历约束图，把解算会受到影响的物体划分在一个island间，不会影响的物体在不同的island中，不同island可以并行。使用bIsSleeping设置island是否处于休眠状态，来决定island是否参与后续的解算。

  ![](pictures\CreateConstraintGraph.png)

FPBDIslandGroupManager:SolveGroupConstraints()：解算约束，包括ApplyPositionConstraints()、ApplyVelocityConstraints()、ApplyProjectionConstraints()。在ApplyPositionConstraints()时，会调用ApplyFrictionCone()对摩擦进行处理，当超过静摩擦(static friction cone)会clip to动摩擦(dynamic friction cone)。



物理模块的东西还有好多好多好多没有写，比如：破碎、更详细的碰撞检测流程等，后面详细看了再更~

如有误，还请指正~

写于2022年11月6日
