---
title: 翻译：A Clean Architecture in .Net （未完成）
date: 2017-08-11 00:13:44
tags: [architecture]
---
sync
# 原文

https://medium.com/@stephanhoekstra/clean-architecture-in-net-8eed6c224c50

# 前言

几年前我偶然发现了Robert Martin的一个关于分离关注点的演讲，在这个演讲的启发下，我尝试把在ASP.NET MVC应用中实践Robert提到的观点。

## 老的方法

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

我意识到，（直接把业务逻辑映射为具体实现的做法，）数据库通常会成为瓶颈（而且难以替换），表现层逻辑混杂着业务逻辑和数据访问代码，这使得（单元/集成）测试变得非常困难。

## 另一种方法

然后我看到了Uncle Bob的演讲Agility and Architecture （[youtube](https://www.youtube.com/watch?v=0oGpWmS0aYQ)），在演讲里面提到了另一种方法："Clean Architecture"。在这种架构下，业务逻辑处于中心位置，而web和数据库作为实现细节（位于架构外围）。

这个演讲解释了我之前碰到的问题，甚至给出了解决方案。**但是，Uncle Bob并没有提供任何源码！** 为了充分理解该架构，我需要一个小的例子。我接着看了很多Uncle Bob的书和视频课程，从中学到了很多。但还是在试图把这些原则应用到真实项目中时依然很挣扎。

随后我尝试在实践中坚持其中的一些原则，并发现确实效果良好。在这篇博客中我会阐述这些实践细节，并解释遵循的原则对实践的帮助，同时也会分享相应的代码样本。

# Clean Architecture概述

在开始之前，先总结此架构的关键元素。我不会对架构本身做详细地解释，如果你之前没有了解过，请先阅览上面提到的视频，或者Uncle Bob的原博客[The Clean Architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)（[译文](http://www.cnblogs.com/yjf512/archive/2012/09/10/2678313.html)）。

## 依赖规则

{% qnimg the_onion_architecture.png %}

Clean Architecture的目的之一，是通过一种干净（清晰？）的方式，把应用的业务逻辑封装起来。

上图展示了该架构，其中domain和use cases处于“洋葱”的中心。

> Uncle Bob:  
> 用一组同心圆表示软件的不同区域，总的来说，越往内，抽象层次越高。外圈是实现机制（web/database/etc），而内圈是策略（业务知识）。

注意图中的依赖方向总是向内依赖。在实践中，这意味着你需要把描述业务逻辑的代码从具体交付机制（UI是web还是app，数据存放在数据库还是NoSql，等等）中分离出来。

{% qnimg policies_and_mechanism.png %}

这样能确保你的业务规则能独立存在，而不依赖于具体的框架选择和实现。

这样做有以下好处：
- 分离关注点。
- 业务规则代码只描述业务领域，不关注其他。
- 在更换数据存储或web框架时，不需要改变业务逻辑代码。
- 业务规则易于测试。
- 业务规则易于重构。

“向内依赖”规则不仅限制业务规则，其他各处也都遵循此规则。但隔离业务规则是Clean Architecture提供的最有价值的一点。

在实践中应该怎样做到呢？

# 练习

我们开始用一个具体例子来尝试实践这些原则。

## 业务需求

我们将从use cases开始，（先从使用use case描述问题开始，）这是很自然的。通常来说，开始一个项目是为了解决用户的具体问题。（在编写use case的时候，我们考虑）软件需要做什么事情，以及需要和用户如何交互。

{% qnimg funda_logo.png %}

[Funda](http://www.funda.nl/) （作者的雇主）是荷兰的一个在线房地产平台。想象我们在开发一个新的功能，让某栋房子的潜在客户联系对应的房产代理。

在Funda我们使用scrum。我们使用user story描述待实现功能的业务价值。

> **User story: 联系房产代理**  
> 
> 作为顾客，我希望联系意向房产的代理，以了解该房产的更多细节。

然后我们会把user story展开为正式的use-case。

> **Use case: 联系房产代理**
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

通常我们的开发人员并不需要把use case如此正规地形成文档。在本案例中我写成这样，是因为这有助于说明Clean Architecture的某些概念。

注意在use case中，我们只描述用户和系统的交互逻辑，而不会说明任何实现细节（如联系信息会被如何存储，UI如何呈现等）。这个use case可能会实现为Console程序，电话程序或者网站，而use case并不负责对此做解释。这正是问题的关键，use case用于描述“系统做什么”，但并不提及“系统怎么做”（Uncle Bob：业务逻辑不应该了解具体实现。）

## Interactor对象

在Clean Architecture中，每个use case的主流程通常可以映射为一个Uncle Bob称之为Interactor的对象。Interactor对象以操作Domain entity（领域实体）的方式描述用户和系统的交互模型。领域实体指的是应用里面的业务对象，例如本例中的Hourse。

<script src="https://gist.github.com/stephanhoekstra/4e4d639c5a388abfbb2b.js"></script>

<script src="https://gist.github.com/stephanhoekstra/6da67a7ee4418a3da661.js"></script>

Interactor类里有几个点需要注意：

1. 几乎是一比一地翻译了之前定义的use-case。
2. 使用了plain request/response（plain指的是POCO对象，区别于Http Request/Response这种为某种context定义的对象）作为输入/输出，并没有任何ASP.NET的引用。
3. 依赖于IRepository。

上述Interactor类只是按功能规格，简明地描述了系统应该如何运作（以支持该功能需求）。代码非常清晰。

Interactor类接收一个plain request，返回一个plain response。这样做可以把Interactor对象从使用它的应用中解耦出来。Interactor对象只需要知道request/response，不再需要知道其他系统实现细节。

Interactor对象通过Repository（在洋葱图中称之为Gateway）获得entity，对其进行操作，再通过Repository保存修改后的entity。这个依赖顺序是违反了向内依赖原则的，但我们可以引入接口，并通过IoC原则把依赖倒置（以解决这个矛盾）。

{% qnimg repository_ioc.png %}

如上图展示，Interactor对象依赖定义在Entities层的IRepository接口。该接口定义了entity的读/写功能，但不提供具体实现。Interactor对象并不依赖该接口在Interface Adapters层的具体实现。

## Entities（领域实体）

上面我们提到了Entity对象，比如本例中的Hourse，现在我们对Entity做进一步讨论。

> Uncle Bob:  
>  
> Entity封装了企业级的业务规则。 一个entity可以是有方法的对象，或者是一组数据结构和函数，这并不重要，只要确保entity可以被用在企业各个不同的应用中。
>  
> 如果你只是写一个独立的应用（而不需要考虑在企业内共享），那么entity就是该应用的业务对象。Entity封装了上层业务规则，通常来说，它们不会因为外部变动而导致需要修改。
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

如果你希望通过ASP.NET MVC应用来暴露我们已经实现的功能，我们接下去要写的代码将会落在Clean Architecture的Interface Adapter层。

> Uncle Bob:
>  
> The software in this layer is a set of adapters that convert data from the format most convenient for the use cases and entities, to the format most convenient for some external agency such as the Database or the Web. It is this layer, for example, that will wholly contain the MVC architecture of a GUI.
> 
> The Presenters, Views, and Controllers all belong in here. The models are likely just data structures that are passed from the controllers to the use cases, and then back from the use cases to the presenters and views.

Indeed this layer is where most of the web application ‘stuff’ is found.

当我们的客户想通过网站联系房产代理，通常他们需要填写一些HTML表单。

A model binder converts the form-data is converted to a strongly typed ContactAgentRequestMessage and the request is handled by the AgentController:



<script src="https://gist.github.com/stephanhoekstra/116d8ee0992811e0135c.js"></script>

Now we must cross the boundary into the inner layers of the application where “the business logic happens”.

We do this by invoking the Handle method on the Interactor, and it is important to stop and think about what data structure we pass into this method.

> Uncle Bob:  
>  
> Typically the data that crosses the boundaries is simple data structures. You can use basic structs or simple Data Transfer objects if you like. Or the data can simply be arguments in function calls. Or you can pack it into a hashmap, or construct it into an object.
>  
> The important thing is that isolated, simple, data structures are passed across the boundaries. We don’t want to cheat and pass Entities or Database rows. We don’t want the data structures to have any kind of dependency that violates The Dependency Rule.  
>  
> […] when we pass data across a boundary, it is always in the form that is most convenient for the inner circle.

This is an essential insight. Repeat:

“当我们传递的数据需要穿越边界的时候，它总是以便于内圈使用的格式传递的。”

In the ASP.NET world it is tempting to use ‘Entity Framework’ classes, which Visual Studio encourages. And then to use those classes all over the place (as your view models, as your entities, in your model binders, etcetera.)

It would also be tempting to pass these across the boundary between Controller and Interactor, but that would violate the Dependency Rule.

Instead, the inner circle should dictate the shape of the input, which is the case in our example. the ContactAgentRequestMessage is defined inside the business logic layer alongside the interactor it corresponds with. The dependency points inwards, and ‘the web is a detail’.

Finally, the interactor does it’s stuff. After evaluating some validation logic, it saves the data somewhere (invokes an IRepository which has an implementation in the outer layer).

> Uncle Bob:
>  
> Similarly, data is converted, in [the interface adapter layer], from the form most convenient for entities and use cases, into the form most convenient for whatever persistence framework is being used. i.e. The Database.  
> Also in this layer is any other adapter necessary to convert data from some external form, such as an external service, to the internal form used by the use cases and entities.

在本例中我简单地实现了一个假的Repository，所以在测试的时候你不需要安装和配置任何数据库。

<script src="https://gist.github.com/stephanhoekstra/bf6bb78d54cbda50d0cf.js"></script>

在实际项目中，通常会使用ORM框架来实现Repository。我喜欢使用NHibernate，有时候也会考虑使用Dapper。

## Presentation

Finally the interactor returns a ContactAgentResponseMessage message to the controller and the use case has been completed!

This message contains the bare minimum information that the end-user needs. It is a data structure formatted in a way that matches the functional requirements from the use case, and that suits the Interactor.

<script src="https://gist.github.com/stephanhoekstra/2f35dc01c6fc0fa2738b.js"></script>

Of course we want to show something nice to the end-user, so we user a Presenter object to convert the raw ResponseMessage into a ViewModel.

The ViewModel is also a plain data structure, but this one is structured in a way that suits the View.

For example, we might want to show them a thank you message or a friendly error message:

<script src="https://gist.github.com/stephanhoekstra/70805bf654a292dc19f9.js"></script>

Finally, the Controller passes the viewmodel to the viewengine, which renders and returns the result to the end-user.

**That’s it!**

# 结论

The ‘inwards dependency rule’ is a way to keep your business rule unaffected by externalities like web frameworks and databases.

This keeps the business rules clean, easily changable and testable.

You can then build your Web or other type of applications on top of this foundation to deliver the system to your user in a prefered way.

I have provided an example of a console application which demonstrates how you can do this in .Net.

Thanks to Uncle Bob, Rodi Evers and Michiel van Oosterhout for inspiration!

You can find a working example including all source code referenced in this article on github.