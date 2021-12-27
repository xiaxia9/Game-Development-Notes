在UE4中，驱动移动的方式可以分为两类：

- 一类是Code driven。胶囊体推动，播放的动画仅做表现层，并不对移动结果产生影响。由于动画和移动各行各的，所以当移动模型和动画不匹配的时候，会产生滑步现象。（详细同步流程请参考上篇文章）
- 一类是Data driven。从动画中提取位移，然后用动画中的位移驱动移动，也就是本篇要介绍的内容——RootMotion。该方式会让游戏表现更自然。

首先需要阅读官方文档：https://docs.unrealengine.com/4.27/en-US/AnimatingObjects/SkeletalMeshAnimation/RootMotion/，对RootMotion有一个大概的了解。

由于Root Motion From Everything目前不支持网络同步，所以下面的内容主要介绍Root Motion From Montages Only这个模式。

# 1 流程

Montage的同步主要由上层处理，比如UAbilitySystemComponent中就提供了一定的支持。也可以使用RPC的方式播放montage。

流程：客户端从动画中提取位移并转换成Velocity，之后调用StartNewPhysics()进行模拟。

![](【UE4】一图看懂移动同步（下）.png)

有些游戏使用MotionMatching技术时，在做网络同步时支持了Data Driven方案，所以了解该同步流程是有一定的帮助的。

RootMotionSource相关的内容，有空的话再补充~

最后，如果错误，烦请指正~



参考：

- https://zhuanlan.zhihu.com/p/74554876

