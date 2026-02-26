---
layout:     post
title:      Dynamic Bone
subtitle:   一种轻量化的动画处理方案
date:       2022-03-03
author:     XStrachey
header-img: img/posts/2021-12-1-2D Fuild-bg.jpg
catalog: true
tags:
    - 物理动画
---

# 什么是物理动画

在动画制作中，帧动画是目前常见的一种动画制作方式，如骨骼动画，这种动画的制作是美术直觉的、可控性高而工作繁杂，在一些大量规律现象的视觉表现时并不是一个很好的制作方式。而对于存在可物理建模的规律现象时，采用物理动画会是一个更为合理的选择。

与各帧中对物体的位置和方向实施精准控制相比，物理动画通过少数物理模拟参数、设定一定约束，使用物理规律实现对物体运动的仿真。物理动画的物理与真实物理相比，其所使用的作用力的精准性视具体情况而定，可信度胜过精准度，一些作用力与真实物理内容并无太多关联，而可以被视作使用作用力对约束条件进行建模，从而实现动画设计师所想要的效果。

采用物理动画的优势在于可以将动画设计师从底层细节中解脱出来，让其可以通过关注“高层关系”之间的确定方案的应用实现其想要的运动效果。

Dynamic Bone插件即是一个物理动画应用的示例。

# Dynamic Bone 相关物理动画模型介绍

## 弹性对象的弹簧-质量-阻尼模型

弹簧-质量-阻尼模型是一类较为常见的弹性对象建模方案。其中，较为直观的方法是将对象的顶点作为质点进行建模，且将对象的各变作为弹簧。另外，弹簧的景直长度可定义为原始边长。对象的质量由动画设计人员任意设定，且假定对象的质量相对于对象的顶点呈均匀分布。弹性系数可设置为任意值，且通常在对象上实现为均匀赋值。利用各个弹簧配合某一阻尼器十分常见，进而可以较好地处理最终的运动行为。

当外部作用力作用于对象的特定顶点上时，碰撞作用、重力现象或则通过脚本定义的作用力会使对象顶点相对于其他顶点产生运动；此类偏移将会引入弹力并作用至其他邻接顶点，并反馈至初始顶点，从而产生进一步的位移；并且引入的弹力将沿着弹簧边传播，使得对象富有弹力，在顶点之间产生持续的相对偏移。

## 刚体模拟

在刚体模拟中，各类作用力均可在系统中进行建模。此类作用力可以根据对象间的相对位置（如碰撞）、对象速度（如黏度）或则用户定义的向量场中的对象的绝对位置（如风力）生成。当作用域对象之上时，这一类作用力将根据对象的质量（线性模式下）以及对象的质量分布（角运动模式下）分别引入线性加速度和角加速度。其中，加速度可以视作delta时间步内的积分运算，进而生成对象速度的变化；相应地，速度在delta时间步内的集成运算可以生成对象的位置和方向变化；对象的新位置和速度将产生新的作用力，并在下一个时间步内，该处理过程重复执行。

刚体模拟中的一个常见问题是基于离散时间步的连续处理建模操作，准确度与计算量之间的折中方案始终是一个问题。相比于真实物理学，计算机动画更多地关注离散时间步中的对象运动建模过程、有效事件及其结果，应谨慎处理离散事件采样过程所产生的数值问题。

> 关于数值近似问题的补充说明
> 在大多数刚体模拟过程中，在Δt时间步内加速度保持恒定的假设条件并不正确，实际操作过程中，随着物体位置和速度的变化，其受力也呈现为持续变化状态。这也意味着，加速度在时间步中产生变化，且其变化方式多呈现为非线性状态。需要注意的是，在时间步开始阶段对作用力采样并以此确定全部时间步的加速度并非最佳方案。在使用欧拉积分法时，即时间步开始阶段的加速度乘以Δt，进而步入下一个函数（速度）时，不难发现时间步的大小对准确度产生影响。当使用较大的时间步时，数值近似路径将与理想状态下的连续路径产生较大的偏差。相应地，当采用较小的时间步时，准确性将得到不同程度的提升，但其计算量也会随之增加。

# Dynamic Bone 算法步骤

Dynamic Bone 是一个简单的基于模拟弹簧振子的算法实现树状柔体的物理模拟插件。虽然基于模拟弹簧振子运动的算法实现，但是DynamicBone各节点之间的距离实际上不会发生变化，比起弹簧，父子节点之间的相对运动更接近串联的单摆。
下图是对 Dynamic Bone 插件的算法步骤的高层级略展：

![算法步骤]({{ site.baseurl }}/assets/20220303-Dynamic Bone/Flowchart.jpg)

# 算法说明

## 韦尔莱积分法

Verlet算法要解决的问题是，给定粒子t时刻的位置r和动量p（速度v），得到t+dt时刻的位置r(t+dt)和动量p(t+dt)（速度v(t+dt)。
牛顿运动方程为：

$$v = a(t) = \frac {f(t)} {m}$$

代入到粒子的坐标关于时间步进的Taylor展开式中:

$$r(t + \Delta t) = r(t) + v(t)\Delta t + \frac {a(t)} {2} \Delta t^2 + \frac {1} {3!} \frac {d^3r} {dt^3}\Delta t^3 + \Omicron(\Delta t^4)$$

得：

$$r(t + \Delta t) = r(t) + v(t)\Delta t + \frac {f(t)} {2m} \Delta t^2 + \frac {1} {3!} \frac {d^3r} {dt^3}\Delta t^3 + \Omicron(\Delta t^4)$$

同理：

$$r(t - \Delta t) = r(t) - v(t)\Delta t + \frac {f(t)} {2m} \Delta t^2 - \frac {1} {3!} \frac {d^3r} {dt^3}\Delta t^3 + \Omicron(\Delta t^4)$$

两式相加：

$$r(t + \Delta t) + r(t - \Delta t) = 2r(t) + \frac {f(t)} {m} \Delta t^2 + \Omicron(\Delta t^4)$$

即：

$$r(t + \Delta t) ≃ 2r(t) + \frac {f(t)} {m} \Delta t^2 - r(t - \Delta t)$$

新位置的计算误差为四阶。
对于速度则可以通过中值定理得出：

$$v(t) = \frac {r(t + \Delta t) - r(t - \Delta t)} {2\Delta t} + \Omicron(\Delta t^2)$$

## Velocity-Verlet

$$r(t + \Delta t) = r(t) + v(t) * \Delta t + \frac 1 2 a_n(\Delta t)^2 = r(t) + v(t) * \Delta t + \frac 1 2 \frac {f(t)} {m} (\Delta t)^2$$

$$v(t + \Delta t) = v(t) + \frac 1 2(a_t + a_{t + \Delta t})\Delta t = v(t) + \frac 1 2 \frac {f(t)_{t} + f(t)_{t + \Delta t}} {m} \Delta t$$

Dynamic Bone插件使用的即是这个的特化版本，具体而言，视质量为0.5，视时间间隔为1。

# 算法公式

## 修正本地坐标与本地旋转量

$$UnityPos_{local} = OriPos_{local}$$

$$UnityRot_{local} = OriRot_{local}$$

## 更新骨骼链整体位移及力

### 更新整体位移

$$r(t + \Delta t) = \overrightarrow {\Delta Pos_{currframe}} = UnityPos_{world} - Pos_{head_{world_{previous}}}$$

$$Pos_{head_{world_{previous}}} = UnityPos_{world}$$

整体位移稍后将附加到每个链上节点。

### 更新力

$$\overrightarrow {Gravity_{world}} = \max ({\frac {(M * \overrightarrow {Gravity_{local}}) \cdot \overrightarrow {RefGravity_{world}}} {\| \overrightarrow {RefGravity_{world}} \|}}, 0)$$

$$\overrightarrow {Force_{world}} = \overrightarrow {Force_{world}} + (\overrightarrow {RefGravity_{world}} - \overrightarrow {Gravity_{world}})$$

这里的力计算是非物理正确的。

## 更新各节点仿真世界坐标、前一帧仿真世界坐标

### 坐标更新核心

以下是位置更新部分：

$$x = \begin{cases}
   Pos_{particle_{world_{previous }}} = Pos_{particle_{world_{cur}}}\\
   Pos_{particle_{world_{cur}}} = UnityPos_{world} &\text{if curParticle is Head} \\
\\
   \overrightarrow {\Delta Pos_{previousframe}} = r(t - \Delta t) = Pos_{particle_{world_{cur}}} - Pos_{particle_{world_{privous}}} \\
\overrightarrow {r_{move}} = r(t + \Delta t) * inert = \overrightarrow {\Delta Pos_{currframe}} * invert \\
Pos_{particle_{world_{previous}}} = Pos_{particle_{world_{cur}}} + r_{move} \\
Pos_{particle_{world_{cur}}} = Pos_{particle_{world_{cur}}} + \overrightarrow {\Delta Pos_{previousframe}} * (1 - damping) + \overrightarrow {Force_{world}} + \overrightarrow {r_{move}} &\text{if curParticle is not Head}
\end{cases}$$

对上述“当当前节点不是链首节点”的情况具体进行说明，先移除弹力等杂项，简化可见核心为：

$$Pos_{particle_{world_{previous}}} = Pos_{particle_{ world_{cur}}}$$

$$Pos_{particle_{world_cur}} = Pos_{particle_{world_cur}} + \overrightarrow {\Delta Pos_{previousframe}} + Force_{world}$$

即：

$$r(t + \Delta t) = r(t) + v(t) * \Delta t + \frac 1 2 \frac {f(t)} {m} (\Delta t)^2 = r(t) + v(t) + f(t) ， m = 0.5, \Delta t = 1$$

其中当前帧速度用上一帧位移近似：

$$v(t) = \overrightarrow {\Delta Pos_{previousframe}} = r(t - \Delta t) = Pos_{particle_{world_{cur}}} - Pos_{particle_{world_{privous}}}$$

继而解释下为什么需要对仿真的当前坐标和前一帧坐标记录进行$+r_{move}$操作，这主要是因为游戏循环的实现，我们的帧之间更多的是离散的点关系，因而旧的“当前坐标”并不等于新的现在的“当前坐标”，而需要对其$+r_{move}$，问题是如何解答$+v(t)$不等于新的现在的“当前坐标”的原因，这是因为这里的$+v(t)$加上的是上一帧的位移（因速度默认1，故速度值即位移值），而因为离散，当前帧的位移通常不等于上一帧的位移，所以我们可以概括地说$+r_{move}$是为了确保引擎正确，$+v(t)$则是提供仿真正确的空间。

### 阻尼

接下来我们引入对阻尼、惯性这两个仿真量的考量。
先看阻尼，阻尼机制基于假设阻力线性正比于速度，阻尼将表现为抑制物体的运动趋势，故建模为：

$$v(t) * (1 - damping)$$

### 惯性

上面已经解释了$+r_{move}$的原因，即当前帧与上一帧的引擎正确的位移，而惯性，可以视为物体维持原有状态的趋势，可以建模为对引擎正确的位移的赋值使用程度，以此来表达物体有多大可能性维持在上一帧位置，即我们设定的原有状态的趋势：

$$r_{move} * inert$$

### 弹性

弹性运动，属于节点的自主运动，不受骨骼链外的力的影响，本质可以视为是骨骼链内部力，在模拟上，可以通过引擎正确坐标与仿真坐标进行求差获得向量，该向量表示了节点的回缩方向：

$$\overrightarrow {D_{elasticity}} = UnityPos_{particle_{world_{cur}}} - Pos_{particle_{world_{cur}}}$$

$$Pos_{particle_{world_{cur}}} = Pos_{particle_{world_{cur}}} + \overrightarrow {D_{elasticity}} * elasticity$$

### 刚性

刚性，即物体间维持原有关联的趋势，主要是维持原有相对距离和维持原有相对旋转量，刚性也是节点的自主运动，本质可以视为是骨骼链内部力。

### 距离

对于距离上的刚性，在模拟上需要先计算出节点间的当前相对距离及原始相对距离，并进行相对距离修正：

$$restLen = \| UnityPos_{parent_{world_{cur}}} - UnityPos_{particle_{world_{cur}}} \|$$

$$\overrightarrow {D_{elasticity}} = UnityPos_{particle_{world_{cur}}} - Pos_{particle_{world_{cur}}}$$

$$currLen = \| \overrightarrow {D_{elasticity}} \|$$

在这里，使用了一个Trick，即为了使动态骨骼的弹性动感更好，提供了一个所谓最大允许相对距离代替原始相对距离：

$$maxLen = restLen * (1 - stiffness) * 2$$

该最大允许距离，最大为原始相对距离的2倍，最小为0（感觉不是很合理）。

$$Pos_{particle_{world_{cur}}} = Pos_{particle_{world_cur}} + \overrightarrow {D_{elasticity}} * \frac {\| \overrightarrow {D_{elasticity}} \| - restLen} {\| \overrightarrow {D_{elasticity}} \|} \text{, if currLen > maxLen}$$

刚性与弹性类似，其影响都是使节点向着引擎正确的位置回缩，有所不同的是刚性只在仿真出的当前相对距离大于所允许的最大距离时才发挥作用。

### 旋转量

对于旋转量上的刚性，在模拟上主要是需要先行依据当前节点与父节点的相对位置和初始相对位置，计算得到两个向量，进而获得这两个向量的夹角：

$$\overrightarrow {Dir_{local}} = Pos_{particle_{local_{ori}}} - 0$$

$$\overrightarrow {Dir_{world}} = M * \overrightarrow {Dir_{local}}$$

$$\overrightarrow {Dir_{world_{new}}} = Pos_{particle_{world_{cur}}} - Pos_{parent_{world_{cur}}}$$

其中向量夹角我们用四元数表示，这个夹角就是仿真计算的旋转量：

$$Rot_{world} = Q(\overrightarrow {Dir_{world}}, \overrightarrow {Dir_{world_{new}}})$$

我们需要将该旋转量加入到其原有旋转上：

$$UnityRot_{world} = Rot_{world} * UnityRot_{world}$$

# 算法缺陷与解决

## 缺陷

上述算法本身是没有明显缺陷的，最多是存在精度不足的问题，但一般游戏尺度下该问题并不明显；但是Dynamic Bone插件本身的实现上却存在一个主要问题，其缺陷在于插件对时间量的考虑缺失，这将在帧率不稳定时造成困扰。

正如我们上面提到的其假定时间间隔为1，这将导致坐标更新核心部分的计算不正确，具体而言，坐标更新核心是在LateUpdate中计算，其假定时间间隔为1，本质上是假定帧间隔稳定相等，但帧间隔实际上是很不稳定的。

## 解决

建议的改进是通过计算当前帧与上一帧的时间间隔，以及上一帧与上上帧的时间间隔，并使用两个时间间隔做一个比值，得到更为准确的时间：

$$\Delta T_{cur} = time_{cur} - time_{previous}$$

$$\Delta T_{previous} = time_{previous} - time_{prevPrevious}$$

$$Rot_{T} = \frac {\Delta T_{cur}} {\Delta T_{previous}}$$

并使用该时间比值替代原本的假定时间间隔1（原本的1即假定$\Delta T_{cur} = {\Delta T_{previous}}$）进行坐标更新核心部分计算。

# 关于Dynamic Bone的性能优化

Dynamic Bone已经是一款很老的插件了，在其诞生时，Unity还未推出其Unity Job System + Brust这一套优化利器，至于使用Compute Shader进行并行化处理直到当前在移动端也是不大现实的选择，而Dynamic Bone也就可以说理所当然地、心安理得地，只是一个单线程插件；单线程当动态骨骼数量较少时倒还无所谓，但一旦同时使用的动态骨骼数量过多，其性能消耗将很恐怖。

针对Dynamic Bone插件的这一实现缺陷，市场上涌现出了很多对动态骨骼的性能优化方案，其中比较经典的是西山居对动态骨骼进行性能优化的技术分享方案，即基于 Job System + Brust 进行并行化计算优化的方案：安全验证 - 知乎。其他，还有基于Compute Shader进行并行化计算优化的方案：安全验证 - 知乎。这些方案的共同点都是看到了动态骨骼计算基于骨骼链的单位划分下具备计算独立性，从而使得并行化计算成为了可能，但相比于单线程，如何处理帧之间数据访问的锁定成了新的问题，需要小心解决。

# 关于项目应用

Dynamic Bone是业内常闻的对布料进行模拟的一款性价比较高的插件，这里的性价比无论从插件购买成本，还是说使用难易度、性能消耗，都是比较高的。但这款插件不是没有缺点的，抛除早期版本的算法缺陷（即使是后续所谓修复版本改动也是有点不知所以），以及分散式和单线程难以应对大量使用的问题之外，其难以让人接受的一点就是其约束不够精细、容易穿模，这个是难以胜任对效果具有更高要求的应用场景。甚至有言称其最大的意义在于方便开发者基于其学习、了解物理动画的思维和实现。

Magica Cloth是可以用于对布料进行模拟的后起之秀，其相对于 Dynamic Bone 插件，具有约束更为精确、不容易穿模，且其基于 Unity Job System + Burst 的实现，替游戏开发者完成性能优化工作；这一插件建议可以购入，进行应用、测试，这里是一篇关于 Magica Cloth 的使用步骤说明。

在其他项目的应用经验中，有一句建议是“在战斗场景使用Dynamic Bone，而在展示界面使用Magica Cloth“，这是为何？还待确认。

# 相关插件仓库

- Jobs 并行化动态骨骼
- MagicaCloth

# 一些有趣的物理动画项目和插件

- https://assetstore.unity.com/packages/tools/physics/magica-cloth-160144
- https://github.com/zalo/MathUtilities
- https://github.com/unity3d-jp/UnityChanSpringBone
- https://github.com/OneYoungMean/Automatic-DynamicBone
- https://github.com/EsProgram/uSpringBone
- https://github.com/JUSTIVE/GPU-Cloth-Simulation

# 参考资料

- 《计算机动画与技术》——第7章”物理动画“、B.7物理初探
- 动态骨骼Dynamic Bone算法详解
- verlet算法解读