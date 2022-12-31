本文包括上层流程、参数含义、示例分析、性能等。

# 1. 基本概念

在上一篇文章中已经介绍过，动画表现可以分为三种类型：关键帧（KeyFramed）、动力/动画（Powered/Animated）、无动力/模拟（Unpowered/Simulated）。实现物理跟随动画，需要实现两种控制器，分别是Local Space控制和World Space控制。也提到了三篇GDC。

详见：[【UE5】基于物理的角色动画（基础篇）](https://zhuanlan.zhihu.com/p/568049696)4.3 刚体状态。

UE 5.1的PhysicsControlComponent也是基于上面提到的概念，官方提供的示例工程也把GDC里面的案例实现了。好耶ヽ(✿ﾟ▽ﾟ)ノ

# 2. PCC介绍

本文提到的PCC为PhysicsControlComponent的简称。

## 2.1 上层流程

PCC可以控制管理多个部位，开放了更多的参数，比如：力、扭矩等，可以动态调整，把参数计算好，通过调用ConstraintInstance的接口实现。

PCC使用EPhysicsMovementType来控制动画驱动类型，Static表示不使用动力学也不使用物理模拟控制，Kinematic表示使用动画控制，Simulated表示使用物理模拟控制。

流程：首先调用MakeControl()创建ControlRecord，用于记录目标点以及ControlData；调用MakeBodyModifier()创建Body修改器，用于控制EPhysicsMovementType；运行时通过修改ControlRecord和BodyModifier来控制运动。PCC组件提供了很多接口，都是在上层控制FPhysicsControlRecord和FPhysicsBodyModifier。ControlRecord控制WorldSpace和LocalSpace下的移动表现，BodyModifier控制上面提到的MovementType。

## 2.2 参数含义

只有理解每个参数的含义，才能更好的控制结果。

首先看一下UPhysicsControlComponent代码结构：

- FPhysicsControlComponentImpl：
  - TMap<FName, **FPhysicsControlRecord**> PhysicsControlRecords;【Name的命名格式为`{ParentBoneName}_{ChildBoneName}_{Index}`】
    - FPhysicsControl PhysicsControl;控制的所有信息
      - FPhysicsControlData ControlData;弹簧阻尼等相关信息
      - FPhysicsControlMultipliers ControlMultipliers;FPhysicsControlData的倍数
      - FPhysicsControlTarget ControlTarget;目标点的位置、方向、速度等
      - FPhysicsControlSettings ControlSettings;常规参数，是否开启碰撞等
      - ......
    - FPhysicsControlState PhysicsControlState;约束等
  - TMap<FName, **FPhysicsBodyModifier**> PhysicsBodyModifiers：设置动力学目标点、重力影响、EPhysicsMovementType。【Name的命名格式`{BoneName}_{index}`或`Body_{index}`】

从上面的代码结构中，可以看到里面比较重要的结构体，接下来详细介绍一下这些结构体中的参数具体含义：

1、**FPhysicsControl**：Control结构，控制的Body为SkeletalMesh或StaticMesh。

| 参数名              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| ParentMeshComponent | 将进行驱动的父级Mesh，如果未设置，则表示此次控制在WorldSpace中进行。 |
| ParentBoneName      | 将进行驱动的父级SkeletalMesh的骨骼名或StaticMesh的名称。     |
| ChildMeshComponent  | 要控制的Mesh。                                               |
| ChildBoneName       | 要控制的SkeletalMesh的骨骼名或StaticMesh的名称。             |
| ControlData         | FPhysicsControlData类型，设置运动强度等参数，可在运行时动态修改。 |
| ControlMultipliers  | FPhysicsControlMultipliers类型，ControlData的倍数，通常会被动态修改。 |
| ControlTarget       | FPhysicsControlTarget类型，控制目标移动的位置和方向，可相对于ParentMesh。 |
| ControlSettings     | FPhysicsControlSettings类型，其他常规参数设置。              |

2、**FPhysicsControlData**：使用弹簧阻尼实现控制body朝着目标。

| 参数名              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| LinearStrength      | 驱动线性运动的强度。也可称之为frequency，是一个伪频率值，是`w=2*PI*f`中的f，f值越小，周期时间越长，表示到达目标的时间越长。 |
| LinearDampingRatio  | 线性运动阻尼比。=0时表示无阻尼；=1时表示临界阻尼，会以最平稳的速度，最短的时间达到稳定平衡；>1时为过阻尼；<1时为欠阻尼，达成稳定平衡的时间较久。 |
| LinearExtraDamping  | 附加线性运动阻尼的值。如果LinearStrength为零，却想要添加阻尼，那么就需要此参数。（注：可以阅读ConvertStrengthToSpringParams()函数的实现，很清晰。） |
| MaxForce            | 驱动线性运动的最大力。零表示无限制。                         |
| AngularStrength     | 驱动角运动的强度。同驱动线性运动的强度的作用。               |
| AngularDampingRatio | 角运动阻尼比。同LinearDampingRatio的作用。                   |
| AngularExtraDamping | 附加角阻尼值。同LinearExtraDamping的作用。                   |
| MaxTorque           | 驱动角运动的最大扭矩。零表示无限制，零扭矩。                 |

3、**FPhysicsControlMultipliers**：用于修改FPhysicsControlData中的参数，优点在于：可以对每个轴分开设置（比如：水平驱动物体，然后让其在重力作用下下降）；可以为例子设置默认的FPhysicsControlData参数，然后通过该结构进行缩放控制。

| 参数名                        | 含义                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| LinearStrengthMultiplier      | 驱动线性运动的强度的各方向的倍率。                           |
| LinearExtraDampingMultiplier  | 线性附加阻尼值的各个方向的倍率。                             |
| MaxForceMultiplier            | 驱动线性运动的最大力的各方向的倍率。ResultMaxForce = MaxForce * MaxForceMultiplier。 |
| AngularStrengthMultiplier     | 驱动角运动的强度的倍率。                                     |
| AngularExtraDampingMultiplier | 角附加阻尼值的倍率。                                         |
| MaxTorqueMultiplier           | 驱动角运动的最大扭矩的倍率。ResultMaxTorque = MaxTorque * MaxTorqueMultiplier。 |

4、**FPhysicsControlTarget**：定义目标位置、方向、速度和角速度。可以指定目标速度，也可以自动计算速度。速度会受FPhysicsControlData参数影响。

| 参数名                     | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| TargetPosition             | child body相对于parent body的目标位置。World Space下的parent body为Null。 |
| TargetVelocity             | child body相对于parent body的目标速度。                      |
| TargetOrientation          | child body相对于parent body的目标方向。                      |
| TargetAngularVelocity      | child body相对于parent body的目标角速度（每秒转数）。        |
| bApplyControlPointToTarget | 是否将FPhysicsControlSettings.ControlPoint用作Target Transform以及physical body的偏移。计算公式为TargetPosition = TargetPosition + TargetOrientation * ControlPoint。 |

5、**FPhysicsControlSettings**：常规设置。

| 参数名                              | 含义                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| ControlPoint                        | 相对于Child Mesh的位置。                                     |
| bUseSkeletalAnimation               | 如果为True且Skeletal Animation存在，则目标速度的结果依赖于骨骼动画顶部的速度。可见CalculateControlTargetData()函数。 |
| SkeletalAnimationVelocityMultiplier | 骨骼动画速度倍率。                                           |
| bDisableCollision                   | 约束中的碰撞状态设置。                                       |
| bAutoDisable                        | 在Tick结束时是否需要设置PhysicsControlState.bEnabled为False。 |

6、**FPhysicsControlState**：每个FPhysicsControlRecord都有一个FPhysicsControlState，用来设置基础状态。

| 参数名             | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| ConstraintInstance | 每个FPhysicsControlRecord对应一个约束。                      |
| bEnabled           | 是否开启物理控制。                                           |
| bPendingDestroy    | 是否在下一刻删除该FPhysicsControlState，初始化时设置为false。 |

7、**FPhysicsBodyModifier**：为每个需要控制的BodyInstance都创建一个PhysicsBodyModifier。

| 参数名                     | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| MeshComponent              | 要修改Mesh。                                                 |
| BoneName                   | 要修改的SkeletalMesh的骨骼名或StaticMesh名字。               |
| MovementType               | Body的移动方式。Static表示不使用动力学也不使用物理模拟控制，Kinematic表示使用动画控制，Simulated表示使用物理模拟控制。 |
| CollisionType              | 碰撞类型。                                                   |
| GravityMultiplier          | 重力系数。如果禁用了重力，则不会生效。                       |
| KinematicTargetPosition    | EPhysicsMovementType为Kinematic，Body移动的目标位置。如果设置了bUseSkeletalAnimation，则应用于BoneName对应的骨骼。 |
| KinematicTargetOrientation | EPhysicsMovementType为Kinematic，Body移动的目标方向。如果设置了bUseSkeletalAnimation，则应用于BoneName对应的骨骼。 |
| bUseSkeletalAnimation      | 如果为true，则目标将应用于SkeletalMesh的Root骨骼或BoneName对应的骨骼。 |
| bPendingDestroy            | 是否在下一刻删除该FPhysicsBodyModifier，初始化时设置为false。 |

（建议还是过一遍代码，上层流程也不是很多......(●'◡'●)

## 2.3 示例分析

理解了参数，示例就能看懂了。

首先看一下父类BP_PhysicsControlCharacter流程：

- 设置Profile：调用SetConstraintProfileForAll()设置约束Profile为HingesOnly。
- 设置骨骼名链：设置LimbSetupData（Array，LimbName表示Key，StartBone表示开始骨骼名，用于创建从StartBone到末端的骨骼名链，IncludeParentBone表示骨骼名链是否包含父骨骼名），这里设置的骨骼链为左臂、右臂、头部、左腿、右腿和spine。然后调用GetLimbBonesFromSkeletalMesh()创建约束骨骼名链（Map，Key为LimbName，Value为FPhysicsControlLimbBones，即骨骼名链和SkeletalMeshComponent）。
- 设置ControlRecord：调用MakeControlsFromLimbBones()为四肢创建WorldSpace下的ControlRecord和ParentSpace下的ControlRecord（注：这里的ParentSpace和GDC中提到的LocalSpace是一个概念。）。WorldSpace下设置LinearStrength和 AngularStrength，ParentSpace下设置AngularStrength，其他都是默认参数。
- 设置BodyModifier：调用MakeBodyModifiersFromLimbBones()创建，设置默认MovementType为Static。

1、第一个例子最简单，该例子是控制WorldSpace下的方块运动。每次隔一段事件随机生成TargetPosition和TargetOrientation，之后通过调整ControlTarget的TargetPosition和TargetOrientation、ControlMultipliers、PhysicsBodyModifier的KinematicTarget和MovementType来实现移动。结果是，物理控制的方块会平滑移动到Target，无物理的效果是直接设置目标点。

2、CompoundBox：可使用PhysicsConstraintComponent来设置约束。

3、Driver：双腿设置为Kinematic。然后所有部位开启ParentSpace模拟，Spine开启WorldSpace模拟，控制世界空间的位置。之后设置Multiplier值。

4、HitReact：死亡的时候，设置的是默认Profile，身体受重力影响，所以表现起来躯体比较柔软，如果追求比较好的效果，还是需要修改profile参数。神秘海域中的示例，更自然。普通受击，可以设置冲量强度，调整起来比较方便。需要考虑碰撞。

![剪秋，我的头好痛？(⊙o⊙)？](pictures\Hit1.png)

5、Hanger：左边的角色开启了物理模拟，右边则是普通动画。开启物理模拟的角色也能更好的与环境碰撞产生反应。手部使用的是Kinematic类型，其他部位使用Simulated。手臂设置为WorldSpace控制，腿部则设置为ParentSpace控制。

6、Carry：使用MakeControl将被抱者和抱人者连接起来，将Strength设置得大一些即可实现。

## 2.4 代码流程

TickComponent()流程：

- 首先遍历FPhysicsControlRecord，如果没有约束，则根据FPhysicsControl信息创建一个ParentBody与ChildBody之间的约束；如果有约束，则调用ApplyControl()应用我们记录的FPhysicsControlRecord信息。在ApplyControl()中：

  - 调用SetDisableCollision()设置碰撞是否禁用。
  - 调用ApplyControlStrengths()计算约束需要的Spring、Damping参数并应用给约束。（注：Spring的计算推导可参考小木子大佬的文章：[游戏开发中的阻尼器和阻尼弹簧](https://zhuanlan.zhihu.com/p/461813778) The Half-life and the Frequency章节 ，Damping的计算没看懂，感觉少了点东西.....如果有同学知道，还请告诉我~）

  ![](pictures\Damping.png)

  - 调用CalculateControlTargetData()计算目标位置和目标速度。

- 之后遍历FPhysicsBodyModifier，调用SetInstanceSimulatePhysics()设置不同的运动状态配置。对于Kinematic类型，还需调用ApplyKinematicTarget()应用Modifier，使用KinematicTarget给Body直接设置TargetTransform。然后调用SetShapeCollisionEnabled()为刚体设置碰撞类型。如果开启了物理模拟，还需要计算重力，并给BodyInstance设置重力影响。

然后就交给物理引擎去模拟了。

# 3. 物理资产设置

好的表现是依托于物理资产的。物理资产调整是很复杂的，也直接影响最终效果。这个调整起来也是很麻烦，神秘海域制作团队也在Maya中建立了测试环境。

从示例工程中的物理资产中可以看到，只设置了三种约束类型，分别是：

- RagdollNoDrives：关节设置为Limit。
- NoLimits：不限制关节自由度。
- HingesOnly：铰链（Hinge）约束，只有一个自由度，用于限制膝盖（thigh_x与calf_x之间的约束）和肘部（upperarm_x与lowerarm_x之间的约束）的一个轴（Twist）的常规约束。（人体模型或机器人学中有介绍）。

仔细观察示例，会发现有时候肢体表现得有些柔软。比如：Walker例子中，左臂碰到碰撞物时，左臂旋转的角度太大，肩胛部分还存在扭曲现象。所以，如果需要在项目中使用，物理资产也还需要进一步调整。

这个调整起来好复杂......（在学了在学了.jpg，研究完了再补充叭）

# 4. 性能

在特定情况下使用物理动画，会给游戏带来更多的趣味性，所以如何灵活地控制物理动画的开启关闭以及Profile切换也是一个很重要的问题。

本机测试对比：DebugGame模式下，空地图是120帧左右；PhysicsControl地图是80帧左右。

部分参数设置：

- Solver iterations（解算器的迭代次数）：UE默认 8（position）和1（velocity），可根据项目情况调整，比如星球大战绝地设置的是64（position）和32（velocity）。
- 在同一个布娃娃中禁用关节之间的CCD更好。

把这些例子单独拿出来测试后，问题不大，主要重心还是放在效果优化上。

使用物理动画的目标是，让模拟尽可能稳定，并且还要有尽可能好的表现。所以不能为了让求解器收敛，而在物理求解器上进行大量的迭代，因此要通过一些设置减少这些情况的发生。比如：沿着某个方向减少质量、提高惯性、、减少实体数量等。

# 5. GDC其他例子介绍

由于自己的demo还在制作中以及资源限制，所以暂时无法实现，以后有条件了再实现。所以本节只总结GDC提到的内容。

1、抓取：hip使用Kinematic，躯干等其他部分由Motor驱动的。

2、死亡布娃娃：

- 如果只在死亡动画的最后开启物理，当周围环境狭窄的时候，播放动画时就可能会产生穿模，动画也不自然。
- 解决方案：身体所有部位都参与物理模拟，但是对Hip创建一个新的约束，约束设置在hip和给定的动画目标之间，这个新约束会拖动hip的物理身体去跟随动画位移，并且锁定新约束的自由度，这样能更好得匹配动画。如果遇到障碍物，动画目标依然在向前运动时，检测一下Root的实际位置和动画目标之间的距离，如果新约束无法抵达目标，则让布娃娃进入自由落体模式。

3、特殊移动：在动画姿势里将纯动画运动和Motor驱动的身体运动进行局部空间混合（local space blend）。

- 滑动：混合物理姿态的50%。
- 攀爬：混合物理姿态的40%。
- 风原场景：混合物理姿态的45%。
- 绳索：混合物理姿态的40%。
- 平衡木悬挂：混合物理姿态的50%。

4、Locomotion：

- 起步、停步和转身时，加入物理能够表现出惯性的感觉。

- 只限制手臂在一定范围内自由摆动，当距离目标太远时，会被约束强制带回来。

5、伴随机器人：

- 机器人腿部设置为Kinematic。

- 主角速度很快时，限制机器人的运动，所以需要在机器人头部和动画目标之间添加新约束。当主角移动慢时，约束允许大范围的移动；当主角移动较快时，允许自由移动的距离就会变成接近零，从而确保头部接近动画目标；当主角移动速度过高时，则把限制自由移动的距离设置为0，约束切换到Lock模式。[注：应该可以通过设置MakeControl，然后调整Strength等参数来实现。]
- 任何不需要碰撞检测的物理动画，比如机器人和大部分主角的动画，都会在自己独立的物理场景中进行处理。
- 先运行主角身上的物理动画，然后根据主角动画的结果，再去配置机器人身上的物理动画。

6、水下：手部和腿部添加Damping用于模拟水的阻力。

# 6. 其他

Maya to DC Script Export：该工具可导出常规刚体的主要参数及其约束，Python脚本是一个好选择~脚本优点如下：

- 不用接触资产管理器，默认规律依然存储在骨架上。
- 方便添加和调整更多的参数。
- 不需要重建游戏资产。
- 快速原型制作（copy & paste FTW）。

更好地管理Profile，同一骨骼层次结构的角色，可以共享Profile。（如果能进行Profile文件之间的Blend，那就更好了~）

需要测试场景，需要美术先在DCC中测试并调整好物理资产，然后导入引擎，与物理工程师一起在工程测试场景中调整优化。

需要Debug工具，可以显示当前刚体和动画。

物理参数是相互影响，所以需要花很多时间去调整，请保持耐心~

游戏运行需要注意物理动画和普通动画逻辑顺序。

尽可能支持AnimNotify控制。

支持向前模拟，看看是否即将发生任何碰撞，然后再接上我们的动画。比如：前方有墙的时候，不会撞到墙，或者用手撑着墙。这样看起来会更智能。

问题：

- 物理模拟中提到了PhysicsBlendWeight，注释写的是0表示只使用动画，1表示只使用物理，不知道能不能设置成其他值来做纯动画姿势和物理姿势的融合？
- GDC2018提到的肌肉状态工具？

