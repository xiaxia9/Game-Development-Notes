![](pictures\人类一败涂地.jpg)

人类一败涂地挺有意思的。

# 1. 前言

基于物理的动画（Physically Based Animation）是计算机图形学中的一个有趣的领域，它包括刚体模拟（Rigid Body Simulation）、柔体模拟（Soft Body Simulation）、流体模拟（Fluid Simulation）、粒子系统（Particle Systems）、蜂拥行为（Flocking）、基于物理的角色动画（Physically Based Character Animation）等。

在动画中添加物理，会使动画更丰富，与场景交互也会更真实。本文介绍角色物理动画最常见的实现之一——布娃娃模拟，本文包括概念、驱动物理跟随动画的方式、UE5中的物理动画代码介绍等。

和之前的文章一样，本文依然只介绍骨骼，且为基础篇，所以**有些内容可以适当跳过**~

# 2. 一分钟实现最简单的角色物理动画模拟

开头先实现一个最简单的物理动画模拟，按照Physical Animation / Ragdoll TUTORIAL in Unreal Engine 4 latest version：https://www.youtube.com/watch?v=_Xnx9L6DgfE流程来。

打开角色蓝图，添加PhysicalAnimation组件，并设置Bone的值为pelvis，然后按下图操作：

- 选择所有刚体，可以添加一个Physical Animation Profiles，然后选择Assign，可以在Details面板中，看到PhysicalAnimation配置，这里可以为每个刚体单独配置数据。 

  ![](pictures\PA-1.png)

- 添加PhysicalAnimation组件，然后选择Mesh，修改Collision，修改胶囊体的大小。

  ![](pictures\PA-2.png)

- 然后编写下面的逻辑，这里创建了一个用于WorldSpace中的PhysicalAnimationData。

  ![](pictures\PA-3.png)

运行结果如下所示（效果与配置有关，不代表最终结果）：

![](pictures\Animation-1.gif)

（这个表现...哈哈哈哈，emm...只是为了引出后面的内容...

# 3. 布娃娃物理系统（Ragdoll Physics System）

Ragdoll将物理和动画结合起来，是一种由物理引擎驱动的程序式动画（procedural Animation）。它是多个刚体（rigid）的集合（在骨骼动画系统中，每个刚体通常与骨骼绑定在一起），通过限制骨骼的移动来实现。当角色死亡时，每个joint的运动都受到了限制，表现也会更真实。

# 4 前置概念及介绍

游戏引擎所指的“角色物理”，更精确地说应称为刚体动力学（rigid body dynamics）模拟。

关于物理资产编辑器的介绍，请见官方文档：https://docs.unrealengine.com/5.0/en-US/physics-asset-editor-in-unreal-engine/

## 4.1 刚体（Rigid Bodies）

在理想状态下，刚体在运动中和受力作用后，形状和大小不变。

在UE5中，FBodyInstance继承于FBodyInstanceCore，它描述了刚体，是物理数据描述的基本单位，包括：

- TWeakObjectPtr< UBodySetupCore> BodySetup：共享的资源信息

- CollisionEnabled：碰撞类型
- LinearDamping、AngularDamping：线性阻尼、角速度阻尼
- DOFConstraint：自由度约束
- GetBodyMass()：质量
- GetBodyInertiaTensor()：惯性张量，张量描述了围绕任何给定轴旋转物体的难度，惯性张量越高越稳定。
- UpdatePhysicalMaterials()：更新材质的摩擦、恢复系数等。

UBodySetup继承于UBodySetupCore，它包含与单个资源关联的碰撞信息，单个UBodySetup实例可以在多个FBodyInstance之间共享。它包括：

- BoneName：骨骼名

- PhysicsType：物理类型
  - Default：继承父组件的设置
  - Kinematic：不受物理影响，但可以与物理模拟的物体交互
  - Simulated：使用物理模拟

USkeletalBodySetup继承于UBodySetup，在父类的基础上增加了一个bSkipScaleFromAnimation开放参数。

![UML](pictures\UML-1.png)

选择物理资产编辑器中的body，可以在Details面板看到FBodyInstance含有的参数：

![](pictures\PhysicsAsset-1.png)

## 4.2 约束（Constraints）

无约束的刚体有6个自由度（degree of freedom, DOF）：可以在3个维度上平移，并可以绕3个笛卡尔轴旋转。

在物理引擎中，约束用于模拟物体间的连接情况，如连杆、布娃娃、吊灯等。在一定程度上削减了DOF。

**常见的约束类型**：

- point-to-point：点对点约束，刚体中某个指定的点与其他刚体的指定点对齐，比如球窝关节（ball and socket joint）。
- hinge：铰链约束，将连接物理的刚体的运动约束在某一个轴上，限制关节中的旋转，比如膝盖I（knees）、肘部（elbows）。有限制铰链只能在预设的角度内旋转，无限制铰链则允许物体旋转无线圈。
- ragdoll：身体上的约束，为模仿真实人体的关节运动而特别设计的，并使用约束链提高模拟的稳定性。
- powered constraint：富动力约束，这个将在后文介绍。

骨骼关节的数量通常比布娃娃刚体的数量多，刚体与骨骼之间存在着映射关系，UBodySetupCore中的BoneName即表示刚体与skeletal mesh中的bone相关联。UE5中的刚体与骨骼之间的关系可以在物理资产中的Skeleton Tree中查看：

![](pictures\PhysicsAsset-2.png)

在UE5中，选择Constraint，可以配置右边的参数，对应的类是FConstraintInstance：

![](pictures\Constraint.png)

## 4.3 刚体状态（Rigid Body States）

动画表现可以分为三种类型：

- 关键帧（KeyFramed）：完全由动画驱动。通常禁用刚体上的碰撞，与其他物体碰撞时动画表现不能被修改。
- 动力/动画（Powered/Animated）：由动画驱动，但可以使用脉冲（impulses）或其他碰撞修改动画表现。物理跟随动画。
- 无动力/模拟（Unpowered/Simulated）：不能由动画驱动，只受重力和约束影响。

**神秘海域4的GDC演讲**（https://www.youtube.com/watch?v=7S-_vuoKgR4）中介绍了两种控制器，分别用于Local Space和World Space控制，来实现物理跟随动画。两种控制分别是：

- 富动力约束控制器（Powered Constraint Controller），也称为Motor。
  - 控制Local Space下的Rotation。Rotation Matching。
  - 难以匹配Animation Pose。
  - Havok物理引擎（他们使用的物理引擎）建议在死亡或垂死角色制作动画时使用Motor，因为Motor在Local Space中工作（控制joint，可以产生非常逼真的结果）。
- 刚体控制器（Rigid Body Controller），也称为关键帧控制器（Keyframe Controller）。

  - 控制World Space下的Position。Position Matching。

  - 操纵单个刚体的速度以匹配动画。线性的。

  - Havok物理引擎建议将此控制器用于实时角色，因为它在World Space中工作。

  - 代码如下：（角速度也同样）

    ```c++
    currentLinearVelocity = getRigidBodyLinearVelocity(myRB)
    currentLinearVelocity += Limit(poseLocalAcceleration, localAccelerationLimit) * localAccelerationGain
    currentLinearVelocity += Limit(poseParentAcceleration, parentAccelerationLimit) * parentAccelerationGain
    desiredVelocity = (currentRBPosition - desiredPosePosition) / deltaTime
    
    currentLinearVelocity += (desiredVelocity - currentLinearVelocity) * positionGain
    havokBody->setLinearVelocity(currentLinearVelocity)
    ```

**EA的GDC演讲**（https://www.gdcvault.com/play/1025210/Physics-Driven-Ragdolls-and-Animation）中也介绍了驱动物理跟随动画的三种方式：

- 约束跟随：
  - 控制普通人类的极限，跟随动画有问题，但是互动好。
  - “变壮”之后，跟随动画好一点，但是互动不好。
- 速度跟随：
  - 速度作为辅助：寻找动画希望我们在WorldSpace中到达的物理位置，根据所需时间，计算速度并应用。
  - 另外一种使用速度的方式：查看当前所处的动画姿势，然后查看下一个动画姿势，计算两者之间的delta并应用delta。
- 约束和速度结合使用

**星战的GDC演讲**（https://www.youtube.com/watch?v=TmAU8aPekEo）中也介绍了驱动物理跟随动画的三种方式：

- Motors：布娃娃身体部位之间的关节约束部分，它们使用指定的动画目标来计算之间的局部受力，这个局部受力会驱动body跟随动画目标。
- Velocities：计算在给定帧时间内，将body从一个点移动到另外一个点所需的速度，然后驱动body移向动画目标。在World Space下工作。

- Constraints：在布娃娃的动态身体和动画目标之间创建一个约束，这个约束的参数可以驱动身体跟随动画目标。

神秘海域4的演讲其实可以概括为Rotation Matching和Position matching。在《游戏引擎架构》书中提到了富动力约束（powered constraint， 外部引擎系统（如动画系统）可以间接地控制布娃娃刚体的平移及定向），也就是神秘海域4中的控制器。可以瞅瞅https://github.com/hairibar/Hairibar.Ragdoll这个项目。

结合上面的内容，**驱动物理跟随动画（Drive Physics Follow Animation）**有四种方式，分别是Rotation Matching、Position Matching、Velocity Matching、Constraint Matching。而且这四种方式是相互独立的，所以可以结合起来使用，并且可以使用权重来控制占比。比如：神秘海域中的Motor Controller和Keyframe Controller可以一起使用，EA的速度和约束也可以一起使用。（注：可以说是四种方式，也可以说是两种、三种，这个不重要，重要是里面的方案.....）

## 4.4 碰撞（Collision）

在实现物理动画的时候，有必要对碰撞进行配置，这里简单介绍一下。

打开角色蓝图，点击Mesh，可以看到Collision配置：

![](pictures\Collision.png)

**bGenerateOverlapEvents**：生成重叠事件

**CanCharacterStepUpOn**， ECanBeCharacterBase：撞上的时候是否可以踩上去

- No：角色无法站立其上
- Yes：角色可以站立其上
- Owner：父级决定是否站立其上，默认为true

**碰撞类型（Collision Enabled）**，ECollisionEnabled：

- NoCollision：不会在物理引擎中创建。不能用于空间查询（raycasts, sweeps, overlaps）或模拟（rigid body, constraints）。
- QueryOnly：仅用于空间查询。无法用于模拟。
- PhysicsOnly：仅用于模拟，不能用于空间查询。
- QueryAndPhysics：可以用于空间查询和模拟。

**物体类型（Object Type）**，ECollisionChannel：物体碰撞类型，

**碰撞响应（Collision Responses）**，ECollisionResponse，每个参与碰撞的对象可以分为三类：

- Ignore：可穿透，无事件通知。
- Overlap：可穿透，有事件通知。
- Block：不可穿透，有事件通知。

概括上面的内容，也就是：只有两个物体同时选择了Block，才会不可穿透。两个物体之间有一个Ignore，就是Ignore，无事件通知。

在物理资产界面，选择刚体，Details面板还有一个Use CCD参数，**Continuous  Collision Detection**（CCD/连续碰撞检测）：物体碰撞检测更精确，但是性能会下降。

## 4.5 配置文件（Profile）

物理动画配置文件（Physical Animation Profile）

- 源码及注释已经讲得很清楚了，存储Bone需要的Motor信息。可以查看FPhysicalAnimationProfile类。

  ![](pictures\PA-4.png)

物理约束配置文件（Constraint Profiles）

- 可以查看FConstraintProfileProperties类。

  ![](pictures\PA-5.png)

# 5 UE5 物理动画介绍

## 5.1 动画更新流程

常规动画更新流程：输入状态，进行逻辑处理，控制器更新，最后输出动画姿势。

![](pictures\AnimationUpdate.png)

物理动画更新流程：输入和逻辑处理不变，控制器更新，然后把动画系统中得到的姿势发送给物理系统，告诉物理这是目标姿势，并将此姿势输入到物理模拟中，然后与环境一起求解，最后输出姿势。

![](pictures\PhysicsAnimationUpdate.png)

## 5.2 骨骼物理动画实现方式

在前面也介绍了很多基础概念，UE5中也提供了一些接口。点开上面的角色蓝图，我们可以对胶囊体和USkeletalMeshComponent进行物理设置，可以通过编写代码对UPhysicalAnimationComponent进行物理动画设置。与骨骼有关的是USkeletalMeshComponent和UPhysicalAnimationComponent，在动画蓝图中还有FAnimNode_RigidBody和FAnimNode_AnimDynamics节点，接下来会介绍这四个。

首先调用UWorld::CreatePhysicsScene()创建物理场（FPhysScene_Chaos）。如果组件设置bSimulatePhysics为true，则组件的位置由物理场景更新到游戏世界，否则，使用kinematic更新位置。

### 5.2.1 USkeletalMeshComponent

很多博主也对USkeletalMeshComponent做了很多介绍。

从代码上看，该组件定义了一个`TArray<struct FBodyInstance*> Bodies`，所以该组件可以有多个FBodyInstance。

在游戏开始运行时，首先调用USkeletalMeshComponent::OnCreatePhysicsState()创建物理状态，当bEnablePerPolyCollision为false时，调用InitArticulated()进行物理初始化。在该函数中调用InstantiatePhysicsAsset()，使用物理资产中的数据创建FBodyInstance数组类型的Bodies，具体实现可以看USkeletalMeshComponent::InstantiatePhysicsAsset_Internal()函数。

接下来打开PhysAnim.cpp文件。

- 在游戏运行时，在USkeletalMeshComponent::PostAnimEvaluation()函数中，会调用UpdateKinematicBonesToAnim()根据动画的变换去更新物理数据，之后调用UpdateRBJointMotors()迭代物理中的关节。
- 在UpdateRBJointMotors()中，首先需要将Mesh组件中的UpdateJointsFromAnimation参数设置成true，然后才会继续往下走。在该函数中，也是通过约束去限制物理模拟，最后会调用FConstraintInstance::SetAngularOrientationTarget()设置目标角朝向。所以这个只是在Rotation上做了匹配，单纯用这个组件，可能并不会达到特别好的效果，所以还需要叠加4.4章节中的其他方案。

![](pictures\SkeletalMesh-1.png)

在UE4 Advanced Locomotion System中也有Ragdoll的存在，在角色死亡时会触发Ragdoll。流程是，在Start时调用SetAllBodiesBelowSimulatePhysics()函数，开启Mesh的物理模拟。在Update时，调用SetAllMotorsAngularDriveParams()设置Angular驱动，之后更新Actor的位置（不然的话，Mesh和胶囊体的位置可能会分开）。

SkeletalMeshComponent还提供了很多接口用于Motor驱动，比如：SetAllMotorsAngularPositionDrive()、SetAllMotorsAngularVelocityDrive()、SetAllMotorsAngularDriveParams()、SetNamedMotorsAngularPositionDrive()、SetNamedMotorsAngularVelocityDrive()等。

### 5.2.2 UPhysicalAnimationComponent

该组件用于处理物理动画，代码并不多。官方文档为https://docs.unrealengine.com/5.0/zh-CN/physics-driven-animation-in-unreal-engine/。

在上面的例子已经看到了，需要设置SkeletalMeshComponent，所以其实还是离不开上面的SkeletalMesh组件。

FPhysicalAnimationData内部还有很多参数：

- bIsLocalSimulation：控制驱动器目标是WorldSpace还是LocalSpace。也就是在神秘海域4 GDC中提到的两种控制器。
- OrientationStrength：修正朝向的强度。
- AngularVelocityStrength：修正角速度的强度。
- PositionStrength：WorldSpace下，修正线性位置的强度。
- VelocityStrength：WorldSpace下，修正线性速度的强度。速度匹配。
- MaxLinearForce：修正线性的最大力。
- MaxAngularForce：修正角度的最大力。

UPhysicalAnimationComponent的一些参数：

- StrengthMultiplyer：Active Motor强度系数。
- TArray< FPhysicalAnimationInstanceData> RuntimeInstanceData：运行时数据的FPhysicalAnimationData数据。

- TArray< FPhysicalAnimationData> DriveData：保存每个刚体的FPhysicalAnimationData数据。
- bPhysicsEngineNeedsUpdating：物理引擎是否需要更新。

流程：

- 首先需要调用SetSkeletalMeshComponent()设置Mesh组件，在该函数中会调用InitComponent()，之后调用UpdatePhysicsEngine()，给bPhysicsEngineNeedsUpdating赋值为true。
- 调用ApplyPhysicalAnimationProfileBelow()应用物理数据配置文件，遍历刚体，调用UpdatePhysicalAnimationSettings()，应用PhysicalAnimationData数据，保存到**DriveData**中。然后调用UpdatePhysicsEngine()更新物理引擎（刚体）。

- 调用SetAllBodiesBelowSimulatePhysics()设置要模拟的骨骼。
- 组件Tick：
  - 当bPhysicsEngineNeedsUpdating为ture时会调用UpdatePhysicsEngineImp()，在该函数中会给**RuntimeInstanceData**赋值。
  - 调用UpdateTargetActors()更新位置。调用SetKinematicTarget_AssumesLocked()把RuntimeInstanceData.TargetActor（Kinematic Actor）移动到TargetTransform（动画数据）。

仅仅设置这些参数，效果还不是很好，会有些僵硬，所以还需要添加约束控制（在物理资产界面，添加Constraint Profiles， Constraint Matching）。在SetMotorStrength()函数内，会调用FConstraintInstance::SetAngularDriveParams()设置angular motor参数，会调用FConstraintInstance::SetLinearDriveParams()设置linear motor参数。在这里，可以通过参数传递定义知道，PositionStrength和OrientationStrength为Spring（Stiffness），VelocityStrength和AngularVelocityStrength为Damping， MaxLinearForce和MaxAngularForce为ForceLimit，使用的是物理引擎中的概念。

### 5.2.3 FAnimNode_RigidBody

在动画蓝图中，还提供了FAnimNode_RigidBody这样一个节点，该节点也是用于模拟含有物理资产的SkeletalMeshComponent。该节点用于轻量级物理模拟，模拟角色的SkeletalMesh上的辅助结构的运动，比如：马尾辫，链子或其他骨骼。具体使用可以参照官方文档https://docs.unrealengine.com/5.0/zh-CN/animation-blueprint-rigid-body-in-unreal-engine/。

![](pictures\RigidBody-1.png)

### 5.2.4 FAnimNode_AnimDynamics

AnimDynamics也是一种轻量级的物理模拟方案，比RigidBody节点更轻量，可以让角色的部分骨骼网格体实现基于物理的附属动画。该节点可用于角色的项链、手镯、背包等物品。可以参考官方文档：https://docs.unrealengine.com/5.0/zh-CN/animation-blueprint-animdynamics-in-unreal-engine/。

![](pictures\AnimDynamics-1.png)

## 5.3 参数设置

Edit -> Project Setting -> Engine -> Physics，这里可以对物理进行一些基本的配置，比如迭代次数等：

![](pictures\Setting-1.png)

# 6 后记

物理很有趣！~      (●'◡'●)

在游戏中，加入物理能让我们和周边环境互动得更好，从而提升沉浸感。国外大厂在这方面花了很多时间，做了很多工作，在细节上打磨了很久，也建立了流程管线。

上面的三个GDC演讲里，还有很多干货，比如特殊情况下（攀爬、平衡木行走、伴随机器人）的物理处理、工具流、FreeRegion、SoftRegion、稳定模拟的方案、数值经验等，篇幅有限，就不在此一一介绍了，推荐去看看。EA的GDC演讲中也提到了在一些情况下使用MotionMathing来匹配动画（其实我觉得准确说应该是PoseMatching），然后叠加物理使效果更好，狂喜。

虽然给游戏添加物理，会使游戏更有趣，但是物理参数相互影响，需要反复调整，所以需要耐心，且需要物理工程师和动画师一起配合。在复杂情况下，做好优化变得更困难了。

物理动画，顾名思义，需要对物理、动画都要熟悉，至少对引擎中的物理和动画的相关实现熟悉。而且物理模拟也需要很多基础性知识与原理，想调好用好物理动画，物理、数学要好！

Cascadeur是一款基于物理的动画制作软件，可以辅助动画师制作。



写于2022年9月25日。