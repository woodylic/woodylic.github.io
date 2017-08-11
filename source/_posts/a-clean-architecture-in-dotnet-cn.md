---
title: 翻译：A Clean Architecture in .Net （未完成）
date: 2017-08-11 00:13:44
tags: [architecture]
---

# 原文

https://medium.com/@stephanhoekstra/clean-architecture-in-net-8eed6c224c50

# 前言

几年前我偶然发现了Robert Martin的一个关于分离关注点的演讲，在这个演讲的启发下，我尝试把在ASP.NET MVC应用中实践Robert提到的观点。

## 从前

> PM：客户需要可以在线上订购产品。  
> 我：好的，那么我需要把订单存在某个数据库里面，并且需要在页面上显示出来  
> PM：对了，如果库存没了，是不允许下单的。  
> 我：你应该早点告诉我的... 我现在要添加新列用于过滤。

<!-- more -->

三个月后

> PM：  
> 为什么我们的bug那么多？  
> 为什么我们的网站那么慢？  
> 为什么这个改动要花这么多时间？

我知道这样的抱怨很公平，因为这些问题是我写的代码造成的。

项目总是从简单开始，然后（由于需求变动）逐渐变得复杂，最终变成一个“大泥球”（big ball of mud）。

我意识到，（在目前的实现中）数据库通常会成为瓶颈（而且难以替换），表现层逻辑混杂着业务逻辑和数据访问代码，这使得（单元/集成）测试变得非常困难。

## 另一种方法

然后我看到了Uncle Bob的演讲Agility and Architecture （[youtube](https://www.youtube.com/watch?v=0oGpWmS0aYQ)），在演讲里面提到了另一种方法："Clean Architecture"。在这种架构下，业务逻辑处于中心位置，而web和数据库作为实现细节（位于架构外围）。

这个演讲解释了我之前碰到的问题，甚至给出了解决方案。**但是，Uncle Bob并没有提供任何源码！** 为了充分理解该架构，我需要一个小的例子。我接着看了很多Uncle Bob的书和视频课程，从中学到了很多。但还是在试图把这些原则应用到真实项目中时依然很挣扎。

随后我尝试在实践中坚持其中的一些原则，并发现确实效果良好。在这篇博客中我会阐述这些实践细节，并解释遵循的原则对实践的帮助，同时也会分享相应的代码样本。

# Clean Architecture概述

在开始之前，先总结此架构的关键元素。我不会对架构本身做详细地解释，如果你之前没有了解过，请先阅览上面提到的视频，或者Uncle Bob的原博客[The Clean Architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)（[译文](http://www.cnblogs.com/yjf512/archive/2012/09/10/2678313.html)）。

## 依赖规则

![image](/uploads/the_onion_architecture.png)

Clean Architecture的目的之一，是通过一种干净（清晰？）的方式，把应用的业务逻辑封装起来。

上图展示了该架构，其中domain和use cases处于“洋葱”的中心。

> Uncle Bob:  
> 用一组同心圆表示软件的不同区域，总的来说，越往内，抽象层次越高。外圈是实现机制（web/database/etc），而内圈是策略（业务知识）。

注意图中的依赖方向总是向内依赖。在实践中，这意味着你需要把描述业务逻辑的代码从具体交付机制（UI是web还是app，数据存放在数据库还是NoSql，等等）中分离出来。

![image](/uploads/policies_and_mechanism.png)

这样能确保你的业务规则能独立存在，而不依赖于具体的框架选择和实现。

这样做有以下好处：
- 分离关注点。
- 业务规则代码只描述业务领域，不关注其他。
- 在更换数据存储或web框架时，不需要改变业务逻辑代码。
- 业务规则易于测试。
- 业务规则易于重构。

“向内依赖”规则不仅限制业务规则，其他各处也都遵循此规则。但隔离业务规则是Clean Architecture提供的最有价值的一点。

在实践中应该怎样做到呢？

# 实践



## Interactor对象



## Entities



## ASP.NET MVC集成



## Presentation



# 结论