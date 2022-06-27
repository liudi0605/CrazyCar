# 网络同步方案

## 背景

所谓“物理同步”，宇面上讲就是"带有物理状态对象的网络同步。，严格上来说它并不是一个标准的技术名词，而是大家约定俗成的一个概念。按照我的个人理解，可以进一步解释为“在较为复杂的物理模拟环境或有物理引擎参与计算的游戏里，如何对持有物理状态信息的对象做网络同步”在英文中，我们可以使用Replicate physics simulated objects或者Networked physics来表示类似的概念。

## [问题与解决方案](https://edu.uwa4d.com/lesson-detail/281/1304/0?isPreview=0)
正如所前面解释的那样，物理同步并不是一种特殊的同步方式，而是在物理引擎和网络同步技术共同发展的条件下而诞生的一种综合性解決方案，其核心手段还是我们熟悉的帧同步或者状态同步。使用帧同步技术我们需家的input信息发送出去，然后让另一端的物理引擎根据输入去模拟结果。如果使用状态同步我们则需要本地模拟好数据并把物理位置、旋转等信息发送到其他的客户端，然后其他客户端可以根据情况决定是否再执行本地的物理模拟（如果是快照同步，由于拿到的就是最终的结果，那么就不需要本地再进行模拟了）这样看来，物理同步与常规的同步也没什么本质上的区别，那么为什么他却是一个难题呢？ 我认为原因有以下两点：

* 物理引擎的不确定性
* 在物理引擎参与模拟的条件下，网络同步的微小误差很容易被迅速放大

首先，我们谈谈物理引擎的确定性向题。很不幸，目前所有的物理引擎严格来说都不是确定性的，因为想保证不同平台、编译器、操作系统、编译版本的指令顺序以及浮点数精度完全一致几乎是不可能的。关于物理确定性的讨论有很多，核心问题大致可以归类为以下几点：

1. 编译器优化后的指令顺序
2. 约束计算的顺序
3. 不同版本、不同平台浮点数精度问题(问题1与问题3其文是密切相关的）

如果游戏只是单个平台上发行，市面上常见的物理引擎（Havok、PhysX*(Unity)*、Bullet）基本上都可以保证结果的一致性。因为我们可以通过使用同一个编译好的二进制文件、在完全相同的操作系统上运行来保证指令顺序并解决浮点数精度问题，同时打开引擎的确定性开关来保证约束的计算顺序（不过会影响性能），这也是很多测试者在使用Unity等商业引擎时发现物理同步可以完美进行的原因。当然，这并不是说我们就完全放弃了跨平台确定性的目标，比如Unity新推出的DOTS架构正在尝试解决这个问题（虽然注释里面仍然鲜明的写着“Reserved for future”）。

接下来，我们再来谈谈第二个难点，即网络同步的误差是如何被物理模拟迅速放大的（尤其在多人交互的游戏中）。为了保证本地客户端的快速响应，通常会采取预测回滚的机制（Client-side prediction，即本地客户端立刻响应玩家操作，服务器后续校验决定是否合法）。这样我们就牺牲了事件顺序的严格一致来换取主控端玩家及时响应的体验，在一般角色的非物理移动同步时，预测以及回滚都是相对容易的，延迟比较小的情况位置的误差也可以几乎忽略。然而在物理模拟参与的时候，情况就会变得复杂起来。

假如在一个游戏中（带有预测，也就是你本地的对象一定快于远端）你和其他玩家分别控制一个物理模拟的小车朝向对方冲去，他们相互之间可能发生碰撞而彼此影响运动状态，就会面临下面的问题。

1. 由于网络同步的误差无法避免，那么你客户端上的发生碰撞的位置一定与其他客户端的不同。
2. 其次，对于本地上的其他模拟小车，要考虑是否在碰撞时完全开启物理模拟（Ragdoll）。如果不开启物理，那么模拟小车就会完全按照其主控端同步的位置进行移动，即使已经在本地发生了碰撞他可能还是会向前移动。如果开启碰撞，两个客户端的发生碰撞的位置会完全不同。无论是哪种情况，网络同步的误差都会在物理引擎的“加持”下迅速被放大进而导致两端的结果相差甚远。

其实对于一般角色的非物理移动同步，二者只要相撞就会迅速停止移动，即使发生穿透只要做简单的位置“回滚”即可。然而在物理模拟参与的时候，直接做位置回滚的效果会显得非常突兀并出现很强的拉扯感，因为我们几乎没办法在本地准确的预测一个对象的物理模拟路径。

《看门狗2》的网络模型是基于状态同步的P2P，主控角色预测先行而模拟对象会根据快照（snapshot，即模拟对象在其主控端的真实位置）使用Projective Velocity Blengding做内插值，他们在制作时也面临和上面描述一样的问题。假如两个客户端各控制一个小车撞向对方，由于延迟问题，敌人在本地的位置一定是落后其主控端的。那么就可能发生你开车去撞他时，你本地撞到了他的车尾，而他的客户端什么都没有发生。

所以，首先要做的就是尽量减少不同客户端由于延迟造成的位置偏差，Matt Delbosc引入了一个Time Offset的概念，根据当前时间与Time Offset的差值来决定对模拟对象做内插值还是外插值，有了合适的外插值后本地的模拟对象就可以做到尽量靠近敌方的真实位置。*（内插值的目的是解决客户端离散信息更新导致的突变问题；外插值的目的是解决网络延迟过大或者抖动导致间歇性收不到数据而卡顿的问题，两种方案并不冲突，可以同时采用。）*

而关于碰撞后位置的误差问题，他们采用了Physics Simulation Blending技术，即发生碰撞前开启模拟对象的Rigid Body并设置位置权重为1（快照位置的权重为0），然后在碰撞发生后的一小段时间内，不断减小物理模拟的权重增大快照位置的权重使模拟对象的运动状态逐渐趋于与其主控端，最终消除不一致性，腾讯的吃鸡手游就采用了相似的解决方案。

可能有些朋友会问，如果我不使用预测回滚技术是不是就没有这个问题呢？答案依然是否定的，假如你在运行一个车辆的中间突然变向，而这个操作被丢包或延迟，只要服务器不暂停整个游戏来等待你的消息，那么你本地的结果依然与其他客户端不同进而产生误差。也就是说除非你使用最最原始的“完全帧同步”（即客户端每次行动都要等到其他客户端的消息全部就绪才行），否则由于网络同步的延迟无法避免，误差也必定会被物理模拟所放大。

仔细分析这两款游戏，你会发现他们采用都是“状态同步+插值+预测回滚”的基本框架，这也是目前业内上比较合适的物理同步方案。

除了同步问题，物理引擎本身对系统资源（CPU/GPU）的消耗也很大。比如在UE4引擎里面，玩家每一帧的移动都会触发物理引擎的射线检测来判断位置是否合法，一旦场景内的角色数量增多，物理引擎的计算量也会随之增大，进而改变Tick的步长，帧率降低。而帧率降低除了导致卡顿问题外，还会进一步影响到物理模拟，造成更严重的结果不一致、模型穿透等问题，所以我们需要尽量减少不必要的物理模拟并适当简化我们的计算模型。

## Crazy Car技术方案

1. 使用物理状态同步方案
2. 模拟车使用模拟车指令驱动，模拟车将操作发送给Server，Server同步给主控方
3. 模拟车会定时发送自己的状态给主控方，用来给主控方做模拟车的纠偏
