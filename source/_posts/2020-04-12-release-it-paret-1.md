---
title: 'Release It 读书笔记, 第一部分'
tags:
  - 系统设计
  - 读书笔记
date: 2020-04-12 21:23:39
---


> 系统失效一定会发生。

# 生产环境的生存法则
> 系统稳定性的两个要求:
> 1.  预防那些能够预防的事情
> 2. 系统在整体上能够从任何未曾预料到的重创中恢复过来


> 设计决策和架构决策（这两个应该是一个意思，设计即架构），也是财务决策。
> 设计/架构，本质上是业务问题，或者说是商业问题。

> 两种架构:
> 1. 侧重对系统更高层次的抽象，很难与一些系统中难以处理的细节产生联系 — 象牙塔架构师，这些人享受绝对完美的最终状态。
> 2. 务实的架构师，不仅和程序员接触，也把自己当作程序员。不断思考动态变化，深入每一个难缠的细节来考虑整个架构。

> 务实的架构师所设计的系统，其中每个组件都足以满足当前的负荷。并且，当负荷随着时间的推移发生变化时，架构师知道要替换哪些组件。

架构的最佳实践应该是动态的，警惕那些「无脑 xx 样就可以了」的设计，这个实践往往忽视了业务的状态、团队的状态和执行的人的因素。一个犯过的典型错误就是 rock 层的设计与拆分。

# 案例研究：让航空公司停飞的代码异常
> 如何预防故障呢？常见的思路:
> 1. CodeReview 能否发现？CodeReview 可以传播好的代码实践，可以促进设计在团队内的共识，但它并不能发现所有的故障
> 2. 更多的测试？总有些问题是 case 没有覆盖的，这些问题是只有知道问题的情况下，才能有 case 去覆盖

软件缺陷一定会产生，无法被消灭。问题在于: 如何控制影响范围，以及如何从故障中恢复。

# 让系统稳定运行
> 威胁系统寿命的主要敌人是内存泄漏和数据增长 。

> 编写寿命测试，或者至少测试那些重要的组件。如果这些都不做，生产环境就会成为寿命测试环境。

紧耦合会加速裂纹的蔓延。MQ 是一个常见的裂纹阻断模式。

系统失效可能性的考虑过程中，几个很容易出问题的点: 外部调用、IO 操作、对资源的使用。单体服务可以缩减外部调用的次数，系统失误的原因一下减少了 33%!

# 稳定性的反模式
关键词: 紧耦合、故障扩散、雪崩

蜘蛛图与蝴蝶图: 软件设计中的两大问题，分层、解耦，让随意设置的蜘蛛图变为精心设计的蜘蛛图。什么是随意？「无脑 xxx」很可能就是一个随意的Flag

系统失效会迅速蔓延。::我们有很多次故障都是因为 MySQL 的瞬时连接压力（冲击）造成数据库变慢，稳定性的裂纹逐渐影响到其他服务，最终导致客户端重试，进而引发雪崩::

其他的一些点
* 应该总是监视缓存的命中率，但不幸的是，我们现在并没有这样的基础设施。
* 整点的 cron job，应该随机摆动一个值来避免「一窝蜂」效应。
* 谨慎无限长的结果集。印象中这个坑也搞过我们很多次。

# 稳定性的模式
> 用于错误处理的代码会增强系统的韧性，系统的用户可能不会因此而感激你，系统不发生停机就没人会留意你，但你晚上能睡的更好。