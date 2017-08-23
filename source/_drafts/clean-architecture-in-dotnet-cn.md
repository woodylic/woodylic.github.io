---
title: 翻译：A Clean Architecture in .Net
date: 2017-08-11 00:13:44
tags: [architecture]
---

# 英文原文

https://medium.com/@stephanhoekstra/clean-architecture-in-net-8eed6c224c50

# 前言  
  
几年前我偶然发现了Robert Martin的一个关于分离关注点的演讲，在这个演讲的启发下，我尝试把在ASP.NET MVC应用中实践Robert提到的观点。

## 碰到的问题

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

我承认这样的抱怨很公平，因为这些问题是我写的代码造成的。

项目总是从简单开始，然后（由于需求变动）逐渐变得复杂，最终变成一个“大泥球”（big ball of mud）。

我意识到，（直接把业务逻辑映射为具体实现的做法，）数据库通常会成为瓶颈，表现层逻辑混杂着业务逻辑和数据访问代码，这使得（单元/集成）测试变得非常困难。

## 解决办法

然后我看到了Uncle Bob的演讲Agility and Architecture （[youtube链接](https://www.youtube.com/watch?v=0oGpWmS0aYQ)），在演讲里面提到了另一种解决方法："Clean Architecture"。在这种架构下，业务逻辑处于架构的中心位置，而web和数据库作为实现细节（位于架构外层）。

这个演讲解释了我之前碰到的问题，甚至给出了解决方案。**但是，Uncle Bob并没有提供任何源码！** 为了充分理解该架构，我需要一个具体的例子。我接下去看了很多Uncle Bob的书和视频课程，从中学到了很多。但还是在试图把这些原则应用到真实项目中时依然十分挣扎。

随后我尝试在实践中坚持其中的一些原则，发现效果确实不错。在这篇博客中我会阐述这些实践细节，并解释遵循的原则对实践的帮助，同时也会分享相应的代码样本。

# Clean Architecture概述

在开始之前，我们先回顾一下Clean Architecture的关键元素。我不会对架构本身做详细地解释，如果你之前没有了解过，请先阅览上面提到的视频，或者Uncle Bob的原博客[The Clean Architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)（[译文](http://www.cnblogs.com/yjf512/archive/2012/09/10/2678313.html)）。

## 依赖规则

{% qnimg the_onion_architecture.png %}

Clean Architecture的目的之一，是通过一种干净清晰的方式，把应用的业务逻辑封装起来。

上图展示了该架构，其中领域实体（Entity）和用例（Use case）处于“洋葱”的中心。

> **Uncle Bob:**  
> 
> 用一组同心圆表示不同层级的软件代码，总的来说，越往内，抽象层次越高。最外层的圈代表的是机制级别的系统。最内层的代表的是策略级别的系统。

注意图中的依赖方向指出，**代码依赖只能使由外向内**。在实践中，这意味着你需要把描述业务逻辑的代码从具体交付机制（UI是web还是app，数据存放在数据库还是NoSql，等等）中分离出来。

{% qnimg policies_and_mechanism.png %}
  
这样能确保你的业务规则能独立存在，而不依赖于具体的框架选择和实现。

这样做有以下好处：

- 分离关注点。
- 业务规则代码只描述业务领域，不关注其他。
- 在更换数据存储或web框架时，不需要改变业务逻辑代码。
- 业务规则易于测试。
- 业务规则易于重构。

向内依赖规则不仅限制业务规则，其他各处也都需要遵循。但隔离业务规则是Clean Architecture提供的最有价值的一点。

在实践中应该怎样做到呢？

# 练习

现在开始用一个具体例子来尝试实践这些原则。

## 业务需求

我们将从用例开始，这是很自然的做法。通常来说，开始一个项目是为了解决用户的具体问题。（在编写用例的时候，我们考虑）软件需要做什么事情，以及需要和用户如何交互。

{% qnimg funda_logo.png %}

[Funda](http://www.funda.nl/) （作者的雇主）是荷兰的一个在线房地产平台。想象我们在开发一个新的功能，让某处特定房产的潜在客户联系对应的房产代理。

在Funda我们使用scrum。我们使用User story描述待实现功能的业务价值。

> **User story: 联系房产代理**  
> 
> 作为顾客，我希望联系意向房产的代理，以了解该房产的更多细节。

然后我们会把User story展开为正式的用例。

> **用例: 联系房产代理**
>   
> 数据: 
> - Email（必填，格式必须符合email规范）
> - PhoneNumber（必填）
> - HouseId（意向房产的Id）
>  
> 主流程:
> - 客户发起 “联系房产代理”请求并提供上述数据。
> - 系统验证所有输入数据。
> - 系统把该请求记录下来，并在稍后通知房产代理。
> - 向用户展示处理结果（记录成功）。
>   
> 异常流程: 验证失败
> - 终止处理该请求。
> - 向用户展示错误信息。

通常我们的开发人员并不需要把用例如此正规地形成文档。在本案例中我写成这样，是因为这有助于说明Clean Architecture的某些概念。

注意在用例中，我们只描述用户和系统的交互逻辑，而不会说明任何实现细节（如联系信息会被如何存储，UI如何呈现等）。这个用例可能会实现为控制台程序，电话程序或者网站，而用例并不负责确定实现方式。这正是问题的关键，用例用于描述“系统做什么”，但并不提及“系统怎么做”（Uncle Bob：业务逻辑不需要了解任何外部结构。）

## Interactor对象

在Clean Architecture中，每个用例的主流程通常可以映射为一个Uncle Bob称之为Interactor的对象。Interactor对象以操作领域实体的方式描述用户和系统的交互模型。领域实体指的是应用里面的业务对象，例如本例中的Hourse。

<script src="https://gist.github.com/stephanhoekstra/4e4d639c5a388abfbb2b.js"></script>

<script src="https://gist.github.com/stephanhoekstra/6da67a7ee4418a3da661.js"></script>

Interactor类里有几个点需要注意：

1. 几乎是一对一地翻译了之前定义的用例。
2. 使用了Plain Request/Response（plain指的是POCO对象，区别于Http Request/Response这种为某种context定义的对象）作为输入/输出，并没有任何ASP.NET的引用。
3. 依赖于IRepository。

上述Interactor类只是按功能规格，简明地描述了系统应该如何运作（以支持该功能需求）。

Interactor类接收一个Plain Request，返回一个Plain Response。这样做可以把Interactor对象从使用它的应用中解耦出来。Interactor对象只需要知道Request/Response，不再需要知道其他系统实现细节。

Interactor对象通过Repository（在洋葱图中称之为Gateway）获得实体，然后对其进行操作，再通过Repository保存修改后的实体。这个依赖顺序是违反了向内依赖规则的，但我们可以引入接口，并通过使用依赖倒置原则来处理这种情况。

{% qnimg repository_ioc.png %}

如上图所示，Interactor对象依赖定义在实体层的IRepository接口。该接口定义了实体的读/写功能，但不提供具体实现。Interactor对象并不依赖该接口在接口适配层（Interface Adapters）的具体实现。

## 领域实体

上面我们提到了领域实体，比如本例中的Hourse，现在我们对此做进一步讨论。

> Uncle Bob:  
>  
> 领域实体用于封装公司的业务规则。 一个实体可以是有方法的对象，或者是一组数据结构和函数，这并不重要，只要确保实体可以被用在公司各个不同的应用中。
>  
> 如果你只是写一个独立的应用（而不需要考虑在企业内共享），那么entity就是该应用的业务对象。Entity封装了上层业务规则，通常来说，在外部环境变动的时候，这些Entity对象是不应被改变的。
>  
> 例如，你不会期望Entity会被页面导航，或者安全方面的改动所影响。在任何应用中，操作层的变更都不应该影响Entity层。

如何为业务领域建模是另一个大的话题，我在这里不做太多讨论。不过（在建模完成后），实现是很直接了当的：在本例中，我使用下面的POCO类描述应用的业务对象和业务规则。

<script src="https://gist.github.com/stephanhoekstra/9ed672b696b240fd49d9.js"></script>

在Funda，我们公布房源信息，而客户可以关注对特定房源，这就是我们的业务模型。而使用领域模型，而不是一个完整的web应用，来描述该业务模型，这样的做法非常简洁。保持这部分代码简单明了，在重构系统或添加新功能的时候，能提供更高的灵活度。

我承认这是一个非常简单甚至简陋的领域模型的例子（有人称之为贫血模型），但我希望通过这个例子可以说明：

你确实可以通过POCO类来描述业务应用的运行行为，然后你可以在应用中直接引用它们，从而避免把业务逻辑混淆到处理web，数据库，消息队列或其他“交付机制”（delivery mechanisms）的代码中。

**任务完成!**

{% qnimg clean_arch_console.png %}


{% qnimg suprised_cat.png %}


嗯，真完成了，某种意义上……

我们已经实现了：
- Interactor对象
- Request/Response消息
- 业务领域Entity

依赖它们，你已经可以对项目待解决的问题进行建模，描述以及单元测试。虽然它还不是一个能工作的web应用。我并不是在开玩笑，某种程度上，业务逻辑是可以在实现任何交付机制（web，database，etc.）之前完成的。

对我来说，一点关键的认知是，有了这些类，你可以为你的项目或者准备要解决的问题开始建模了。你可以为这些类写单元测试或者console应用，以证明这些业务逻辑是满足需求的。事实上，我在源码里面确实加入了一个用于测试的console应用。

下面我会简略地谈及如何把这些已有部分集成到web应用中。但我希望现在已经说服你相信，web应用可以搭建在我已经演示的Clean Architecture之上。

## 集成ASP.NET MVC应用

如果你希望通过ASP.NET MVC应用来呈现我们已经实现的业务逻辑，接下去要写的代码将会落在Clean Architecture的Interface Adapter层。

> Uncle Bob:
>  
> 处于Interface Adapter层的代码是一系列适配器，用于把数据从便于use case和业务实体操作的格式，转为便于外部中介（如web或database）操作的格式。
> 
> 例如GUI的MVC结构，presenter、view、controller都是属于这一层。这层很可能是通过controller将数据结构传给use case，并且返回数据给presenter和view。

实际上大部分的web应用具体实现能在这层找到。

当我们的客户想通过网站联系房产代理，通常他们需要填写一些HTML表单。然后**模型绑定器(model binder)**会把表单数据转换为强类型的**ContactAgentRequestMessage**对象，并交由**AgentController**处理该请求：

<script src="https://gist.github.com/stephanhoekstra/116d8ee0992811e0135c.js"></script>

现在我们必须越过边界进入内层，也就是业务逻辑的所在。我们通过调用Interactor对象的Handle方法做到了这点，在这里，我们需要先考虑清楚，在方法调用中，我们应该使用什么样的数据结构传递参数。

> Uncle Bob:  
>  
> 一般跨层调用的数据是简单的数据结构。你可以使用数据结构或者是简单的数据传输流，又或者可以通过函数的参数来进行传递。你也可以将数据封装到一个hashmap结构，或者一个对象中。
>
> 数据传输最重要的事情是无依赖，简单。我们并不希望跨层传递的数据是Entity，或者是数据表的行数据，理由是我们不希望数据有任何形式的违反依赖规则。
>  
> ...
>  
> 当我们跨层传递数据的时候，我们应该以便于内圈使用的数据格式传递。

最后一句是最关键的，重复一下：

“当我们跨层传递数据的时候，我们应该以便于内圈使用的数据格式传递。”

在ASP.NET的世界里，如同Visual Studio的示例，人们倾向（基于设计好的数据库）使用Entity Framework创建实体类，并且把它们用作view model，domain entity，DTO等等。人们也会很自然地使用它们在Controller和Interactor之间传递数据，但这违反了向内依赖规则。

与之相反，内圈应该定义输入参数的数据格式。在我们的例子中，作为Interactor和Controller交互的参数，**ContactAgentRequestMessage**类同样被定义在了业务逻辑层中。这样做就遵循了向内依赖规则。

最后，Interactor对象完成它自己的工作。执行一系列的验证逻辑，然后把数据保存下来（保存动作通过IRepository接口执行，而具体的保存工作由外圈的IRepository实现来完成）。

> Uncle Bob:
>  
> 类似地，在Interface Adapter层中，数据会从便于Entity和Use Case使用的结构，转换为便于持久层（比如数据库）使用的结构。
> 
> 同样地，在这层里面包含了其他必要的适配器，用于把数据从外部结构转换为便于use case和entity使用的内部结构。

在本例中我简单地实现了一个假的Repository，测试的时候你不需要安装和配置任何数据库。

<script src="https://gist.github.com/stephanhoekstra/bf6bb78d54cbda50d0cf.js"></script>

在实际项目中，通常会使用ORM框架来实现Repository。我喜欢使用NHibernate，有时候也会考虑使用Dapper。

## 表现层

在业务逻辑处理完成后，Interactor对象将返回一个ContactAgentResponseMessage消息给Controller，use case的任务到此结束。消息对象只包含最终用户必须的信息，它实际上是一种数据结构，符合use case描述的功能需求，适合被Interactor调用。

<script src="https://gist.github.com/stephanhoekstra/2f35dc01c6fc0fa2738b.js"></script>

Of course we want to show something nice to the end-user, so we user a Presenter object to convert the raw ResponseMessage into a ViewModel.

然后我们希望以友好的界面展示给用户，我们会使用Presenter对象把原始的ResponseMessage换行为ViewModel。

ViewModel也是一种简单数据结构，但它的结构是为适配View而设计的。

例如，我们希望展示友好的操作成功或失败的消息：

<script src="https://gist.github.com/stephanhoekstra/70805bf654a292dc19f9.js"></script>

Finally, the Controller passes the viewmodel to the viewyingqin, which renders and returns the result to the end-user.

最后，Controller把ViewMode传递给视图引擎，引擎将用于渲染视图并展示给用户。

**任务完成!**

# 结论

遵循“向内以来规则”可以避免业务逻辑被外包框架和实现的变更所影响。这样做可以保持业务逻辑独立整洁，易于测试和修改。

在这个（业务实现的）基础上，你可以根据用户的需要，选择以web或者其他类型的应用方式来交付系统。

我提供了一个console应用作为例子，展示在.Net中如何实践这个理论。你可以在我的github上找到该例子，其中包含本文展示的所有源码。

感谢Uncle Bob，Rodi Evers和Michiel van Oosterhout提供灵感！