---
layout: post
title:  "实现领域驱动设计 - 学习笔记"
date:   2019-02-27 21:45:30 +0800
categories: pattern
---

* TOC
{:toc}


## 第 0 章: DDD 总览

- **通用语言(Ubiquitous Language)** 作用于某个 **限界上下文(Bounded Context)**，不管在战术上还是战略上设计软件模型，你都应该保证：在一个特定的限界上下文中只使用一套通用语言，并且保证它的清晰性和简洁性。

### 战略建模

限界上下文是一种概念上的边界，领域模型工作于其中。

在战略设计中，你会发现**上下文映射图(Context Map)** 是非常有用的。

### 架构

有时，一个新的限界上下文或上下文映射图可能需要一种新的**架构(Architecture)**。

你应该牢记：通过战略设计和战术设计而成的领域模型应该是**架构中立**的。

### 战术建模

我们在限界上下文中进行战术建模。

战术设计的一个重要模式是**聚合(Aggregate)**。

- **聚合(Aggregate)**可以由单个**实体(Entity)**组成，也可以由一组实体和**值对象(Value Object)**组成，此时我们必须保证在聚合的整个生命周期中保持事务上的一致性。
- 聚合实例通过**资源库(Repository)**进行持久化，另外，对于聚合的查找和获取也通过资源库完成。
- 在领域模型中，有些业务操作不能自然地放在实体或值对象上，此时我们可以使用无状态的**领域服务(Domain Service)**，它可以进行跨聚合的操作。
- **领域事件(Domain Event)**表示领域模型中发生的重要事件。在对聚合进行命令操作时，聚合本身将发布领域事件，通知订阅方进行处理。
- **模块(Module)**可以简单看成是 Java 中的包或 C# 中的命名空间，模块中包含的领域对象应该是内聚在一起的。

## 第 1 章: DDD 入门

DDD 作为一种软件开发方法，它主要关注以下几个方面:
- DDD 将领域专家和开发人员聚集到一起，这样所开发的软件能够反映出领域专家的思维模型。领域专家将和开发人员一起创建一套适用于领域建模的通用语言。通用语言有助于促使原本存在分歧的领域专家们达成一致意见。
- DDD 关注业务战略。它帮助我们定义不同团队之间的组织关系。DDD 的战略设计用于清除地界分不同的系统和业务关注点，这样可以保护每个业务层面的服务。更进一步，这将指导我们如何实现**面向服务架构(Service-oriented architecture)**或**业务驱动(business-driven architecture)**。
- 通过使用战术设计建模工具，DDD 满足了软件真正的技术需求。这些设计工具使开发人员能够按照领域专家的思维模型开发软件。同时，所开发出来的软件是可测试的，能够尽量避免错误，能执行**服务层面协议(Service Level Aggrement, SLA)**，具有很好的伸缩性，并且允许分布式计算。

### 贫血症和失忆症

- **贫血领域对象(Anemic Domain Object)** 描述的是一个缺少内在行为的领域对象。贫血领域对象是不好的，比如，由于存在**对象-关系阻抗失配(Object-Relational Impedance)**，开发者需要将很多时间花在对象和数据存储之间的映射上。

### 如何 DDD

通用语言和限界上下文同时构成了 DDD 的两大支柱，他们是相辅相成的。

### 使用 DDD 的业务价值

不管使用什么技术，我们的目的都是提供业务价值。DDD 的业务价值大致总结为以下几点：
1. 你获得了一个非常有用的领域模型；
2. 你的业务得到了更准确的定义和理解；
3. 领域专家可以为软件设计做出贡献；
4. 更好的用户体验；
5. 清晰的模型边界；
6. 更好的企业架构；
7. 敏捷、迭代式和持续建模；
8. 使用战略和战术新工具；

### 实施 DDD 所面临的挑战

DDD 常见的挑战：
- 为创建通用语言腾出时间和经历；
- 持续地将领域专家引入项目；
- 改变开发者对领域的思考方式；

业务领域的类型本身并不自动地决定应该选择哪种开发方式。你的团队应该考虑一些重要的问题，然后做出决定。请考虑一下因素：
- 团队是否有领域专家，如果有，你如何围绕领域专家组织自己的团队？
- 虽然目前来说，你的业务领域是简单的，但它将来会变得复杂吗？对于复杂的系统来说，使用事务脚本是存在风险的。当领域变得复杂时，是否可能将系统重构到富含行为的领域模型？
- DDD 的战术模式是否可以简化与其它限界上下文的继承，不管是第三方的开始定制开发的？
- 使用事务脚本是否的确可以减少代码量？(经验表明，不管哪种开发方式，事务脚本都不能减少代码量)
- 你项目的进度安排是否允许在战术模式上有所投入？
- 在核心域上的战术投入能否消除架构变化带来的影响？事务脚本是做不到这一点的(和领域模型层相比，其它层更容易受到架构变化的影响)。
- 客户是否的确能从这种持续设计和开发的方式中获益，或者有现成的产品就能满足他们的需求？换句话说，我们是否应该一开始就考虑定制化开发？
- 使用 DDD 的战术开发模式会比其他开发方式更加困难吗，比如事务脚本？(这个问题很大程度上取决于团队成员的技能水平和是否有领域专家)。
- 如果团队已经具备了实施DDD的条件，我们还会刻意地选择另一种开发方式吗？有些开发者已经将模型的持久化变得很实用了。

DDD并不笨重：
DDD也倾向于“测试先行，逐步改进”的设计思路。在你开发一个新的领域对象时，比如实体或值对象，你可以采用以下步骤进行：
1. 编写测试代码以模拟客户代码是如何使用该领域对象的；
2. 创建该领域对象以使测试代码能够编译通过；
3. 同时对测试和领域对象进行重构，知道测试代码能够正确地模拟客户代码，同时领域对象拥有能够表明业务行为的方法签名。
4. 实现领域对象的行为，直到测试通过为止，再对实现代码进行重构；
5. 向你的团队成员展示代码，包括领域专家，以保证领域对象能够正确地反映通用语言。

### 虚构的案例，真实的实践

- 一个公司叫 SaaSOvation, 该公司旨在开发一系列 SaaS 产品。
- 该公司计划先后开发两套产品：
    - 旗舰产品名为 CollabOvation, 是一套企业协作软件，并且加入了社交网络的功能。该产品的功能包括论坛、博客、即时消息、留言板、文档管理、通知、提醒、活动跟踪和 RSS 等。所有这些都是为了满足企业业务的需求，帮助他们在项目中提高工作效率。
    - 第二套产品名为 ProjectOvation, 这是该公司的开发团队主要关注的核心域。该产品主要用于敏捷项目的管理，使用 Scrum 项目管理模型，其中包括产品、产品负责人、团队、待定项(backlog item)、计划发布(planned release)和冲刺(sprint)对待定项的评估通过对业务价值的分析来确定。
- CollabOvation 和 ProjectOvation 并不是两套互不相关的产品。SaaSOvation 公司非常看重敏捷软件开发过程中的团队协作。因此 CollabOvation 可以作为 ProjectOvation 的增值服务。
- 首先启动的项目是 CollabOvation。在项目组的早期会议中，他们决定采用 DDD，然而他们使用的其实是 DDD-Lite。
- DDD-Lite 是 DDD 战术模式的一个子集，它并不强调对通用语言的使用。此外 DDD-Lite 也忽略了限界上下文和上下文映射图。更过的，DDD-Lite 是关于技术实现层面的，虽然这也有好处，但好处并没有与 DDD 战略模式一起使用时那么大。
- 更糟糕的是，虽然 SaaSOvation 避开了 DDT-Lite 的一些陷阱，但这纯属侥幸，因为他们的两套核心产品自然地形成了各自的限界上下文。这也使得 CollabOvation 模型和 ProjectOvation 模型得到了正式地分离，但这也是偶然的，并不意味着团队就是了解限界上下文的，而这也是为什么该团队在一开始就遇到了问题的原因。


## 第 2 章: 领域、子域和限界上下文

### 总览

从广义上讲，**领域(Domain)**即是一个组织所做的事情以及其中所包含的一切。每个组织都有它自己的业务范围和做事方式这个业务范围以及在其中所进行的活动便是领域。当你为某个组织开发软件时，你面对的便是这个组织的领域。

在 DDD 中，一个领域被划分为若干**子域(SubDomain)**，领域模型在**限界上下文(Bounded Context)**中完成开发。

试图创建一个全功能的领域模型是非常困难的，并且很容易导致失败，对领域的拆分将有助于我们成功。

既然领域模型不能包含整个业务系统，我们应该如何来划分领域模型呢？

#### 工作中的子域和限界上下文

对于如何使用子域，可以先看一个非常简单的例子：

![]({{site.url}}/asset/implementing-ddd-market-sample.jpg)

- 零售商需要展示产品，允许买家下单和付款，还得安排物流。
- 零售商的领域可以分为 4 个主要的子域: 产品目录、订单、发票、物流。图中的上半部分表示了这样一个电子商务系统。
- 如果要向以上的电子商务系统中再加入一个库存(Inventory)系统，情况会怎么样？
- 在上面的电子商务系统中，我们可以找出多个隐式的领域模型，可惜的是它们并没有很好地被分离出来，随着各个模型中不断加入新的功能，它们之间的复杂关系对于每一个模型都将是阻碍。
- 通过 DDD 战略设计工具，我们可以按照实际功能将这些交织的模型划分成逻辑上相互分离的子域，从而在一定程度上减少系统的复杂性。逻辑子域的边界在上图中用虚线表示(虽然实际上没有这样划分)。
- 为了能够灵活地调整库存、根据准确的需求囤积产品，从而减少成本，零售商可以采用一个预测引擎，来优化库存系统。图中的第三个限界上下文就是一个外部的预测系统。
- 订单子域和库存限界上下文向预测系统提供历史销售数据，产品目录子域提供全局的产品条形码，这有助于预测系统在全球范围内对产品的销售情况进行比较。
- 子域不一定要做得很大，并且包含很多功能。有时，子域可以简单到只包含一套算法，这种简单的子域可以以模块的形式从核心域中分离出来，而不需要包含在笨重的子系统组件中。
- 需要注意的是，一个限界上下文并不一定只包含在一个子域中，但这是可能的。例如图中的库存限界上下文包含在了一个库存子域中。
- 电子商务系统在开发的时候没有正确地采用 DDD，在这个系统中，我们识别除了 4 个子域。库存系统这种清晰的模型有可能是采用了 DDD 的结果。
- 在上图中，哪种类型的限界上下文在语言表达层面设计的更好呢？在电子商务系统中，有一些术语在子域中时存在冲突的，比如“顾客”这个术语可能有多重含义，在浏览产品目录的时候“顾客”是一种意思，在下单的时候“顾客”又表示另一种意思。在浏览目录时，“顾客”被放在了先前购买情况、忠诚度这样的上下文中，而在下单时，“顾客”的上下文包括名字、地址、订单总价等一些付款术语。
- 从表面上看来，我们可以认为库存系统比电子商务系统具有更高的 DDD 健康指数，它比电子商务系统模型更容易集成。
- 上图进一步表明，一个企业的限界上下文肯定不是孤立存在的，不同的子域之间的实现表示了它们的集成关系，这表明不同的模型需要相互协作。我们将在 **上下文映射图** 中学到不同的集成方案。

#### 将关注点放在核心域上

以下是一个抽象试图，用以表示一个领域：

![]( {{site.url}}/asset/implementing-ddd-domain-abstract-sample.jpg )

- 上半部分是核心域，也就是业务成功的主要促成因素。战略层面上，应该给核心域最高的优先级，配备最好的领域专家和最优秀的开发团队。
- 另一种子域叫做支撑子域，这些子域用于支撑我们的业务，但却不是核心。它跟通用子域的区别在于它专注于业务的某个方面。
- 如果一个子域被用于整个业务系统，那么这个子域便是通用子域。
- 子域和限界上下文有相交的地方，这并不是什么坏事，因为这正是企业级软件的本来面目。

### 战略设计为什么重要

SaaSOvation 公司因为没有重视 DDD 战略设计，只关注实体、值对象这些战术设计，导致他们将两个模型创建成了一个：

![]( {{site.url}}/asset/implementing-ddd-why-strategy-is-so-important.jpg )

问题在于，论坛(Forum)和讨论(Discussion)等概念与错误的语言概念耦合起来了。用户(User)和权限(Permission)与 CollabOvation 产品中协作活动没有任何关系，并且与 CollabOvation 的通用语言也风马牛不相及。用户和权限是与身份(Identity)和访问(Access)相关的概念，它是与安全(Security)相关的。与 Collaboration 上下文没有任何关系。这里应该注意的是 Collaboration 上的概念，比如作者(Author)和主持者(Moderator)，这才是 CollabOvation 中的正确概念和语言。

我们以 “模型名+上下文” 的方式来命名限界上下文。对 SaaSOvation 公司来说，他们的产品有 协作上下文(CollabOvation 产品中的协作模型)、身份与访问上下文(一个新的上下文)、敏捷项目管理上下文(ProjectOvation 产品中的项目管理模型)。

以上讨论体现了战略设计中，通用语言和限界上下文的作用。

### 现实世界中的领域和子域

领域中同时存在 **问题空间(problem space)** 和 **解决方案空间(solution space)**。在问题空间中，我们思考的是业务所面临的挑战，而在解决方案空间中，我们思考如何实现软件以解决这些业务挑战。
- 问题空间是领域的一部分，对**问题空间的开发将产生一个新的核心域**。评估问题空间时还应该考虑已有子域和额外所需子域。因此问题空间时核心域和其它子域的组合。
- 解决方案空间包括一个或多个限界上下文，即一组特定的软件模型。这是因为**限界上下文即是一个特定的解决方案**，它通过软件的方式来实现解决方案。

通常我们希望将子域一对一的对应到限界上下文(即一个问题由一个特定的解决方案来处理)。如之前零售系统的例子而言，可能有一些遗留系统，其中子域和限界上下文存在相交的地方。但我们可以通过新的努力来将它们分离开来。

为了澄清问题空间和解决方案空间的区别，让我们来看一个例子。

在已有的 ERP 系统的基础上，某企业计划开发一个特定的领域模型来降低成本，该模型可以为采购人员提供决策工具，通过一些算法来实现一些工作的自动化。这种工具可以大大提高该企业的竞争优势。对这种问题空间的开发能够产生一个新的核心域。为了保证你的项目朝着正确地方向行进，你需要先回答以下问题：
- 这个战略核心域的名字是什么，它的目标是什么？
- 这个战略核心域包含哪些概念？
- 这个核心域的支撑子域和通用子域是什么？
- 如何安排项目人员？
- 你能组建出一支合适的团队嘛？

经过一些考虑，我们得出了一些结论，我们将开发一个名为 "最优获取" 的新的核心域，它将通过 "最优获取上下文" 这一限界上下文来解决对应的问题：

![]( {{site.url}}/asset/implementing-ddd-problem-space-and-solution-space.jpg )

- 在上图中，用于捕获决策工具和算法的采购模型表示了核心域的解决方案。该领域模型将在 "最优获取上下文(Optimal Acquisitions Context)" 中实现，该上下文与 "最优获取核心域(Optimal Acquisitions Core Domain)" 存在一对一的关系。
- 除了最优获取上下文，还有 "采购上下文(Purchasing Context)"。该上下文用于辅助最优获取上下文，以改进采购过程中的技术细节。ERP 系统本身就有采购模块，这个采购上下文只是简化了最优获取上下文和 ERP 的交互，也即是一个用于操作 ERP 接口的便捷模型。这个新的采购上下文和先前的 ERP 采购模块同属于采购(支撑)子域。
- 整个 ERP 模块是一个通用子域，你甚至可以购买另外的采购系统来替换该子域。
- 最优获取上下文还需要和 "库存上下文(Inventory Context)" 进行交互。库存系统使用了 ERP 的库存模块，该模块位于库存子域中。
- 为了给货递合同方提供方便，库存上下文还是用了第三方的地图服务。当库存上下文需要使用外部的地图上下文(Mapping Context)时，他们之间的数据交换需要一定程度的翻译才行，所以在库存子域中有一个映射上下文。

### 理解限界上下文

限界上下文是一个显式的边界，领域模型便存在于这个边界之内。创建边界的原因在于，每一个模型概念，包括它的属性和操作，在边界之内都有特殊含义。

限界上下文并不旨在创建单一的项目资产，它并不是一个单独的组件、文档、或者框图。因此，它并不是一个 JAR 或者 DLL，但是这些可以用来部署限界上下文。

在很多情况下，在不同模型中存在名字相同或相近的对象，但是他们的意思却不同。

当需要集成时，我们必须在不同的限界上下文之间进行概念映射。

#### 限界上下文不仅仅只包含模型

一个限界上下文并不是只包含领域模型。它通常标定了一个系统、一个应用程序或者一种业务服务。

当模型驱动着数据库 Schema 的设计时，此时的数据库 Schema 也应该位于该模型所处的上下文边界之内。

系统中有可能存在诸如 Web 服务(Web Services)之类的组件。我们也可以使用 REST 资源与模型交互，此时的 REST 资源即被称为 **开放主机服务(Open Host Service)**。或者，我们可以使用 SOAP 或消息服务端点。在以上所有情况中，哪些面向服务的组件都应该位于上下文边界之内。

#### 限界上下文的大小

如果我们的模型是音乐，那么它应该表现出的是完整性、纯洁性、力量、优雅和美。其中的 "音符" —— 模块、聚合、事件和服务 —— 的数量正好是设计所要求的那么多。模型的听众不会问及到像 "为什么中间会有一些奇怪的音符？" 这样的问题。同时，它们也不会因为丢失了某些 "音符" 而感到不解。

#### 与技术组件保持一致

将限界上下文想成是技术组件并无大碍，只是我们需要记住： 技术组件并不能定义限界上下文。

当使用 IDE 时，比如 Eclipse 或 IntelliJ IDEA，一个限界上下文通常就是一个工程项目。

当使用 Visual Studio 和 .NET 时，在同一个解决方案中将用户界面、应用服务和领域对象分离在不同的子项目中是合理的。

### 限界上下文示例

我们的 SaaSOvation 团队所选择的 3 个限界上下文最终将与各自所对应的子域形成一对一的关系。如下图所示：

![]( {{site.url}}/asset/implementing-ddd-context-sample.jpg )

#### 协作上下文

即解决 协作问题 的限界上下文，可能是一套软件。

负责设计和实现 *协作上下文* 的核心团队需要在第一次软件发布中包括以下功能：论坛、共享日历、博客、即时消息、wiki、留言板、文档管理、通知与提醒、活动跟踪和 RSS 订阅。

![]( {{site.url}}/asset/implementing-ddd-sample-collab-context.jpg )

在项目启动时，SaaSOvation 团队只使用了 DDD-Lite，他们错误地将安全和权限相关的逻辑引入了协作模型中，将两个模型混合在了一起。在实现代码中，他们在核心业务逻辑中去检查客户的权限：

```java
/**
 * 论坛(Forum)实体
 */
public class Forum extends Entity {
    // ...

    /**
     * 发起话题讨论
     * 
     * @param userName 用户名
     * @param subject 标题
     */
    public Discussion startDiscussion (String userName, String subject) {
        if (this.isClosed()) {
            throw new IllegalStateException("Forum is closed.");
        }

        // 检查该用户是否有权限发起讨论
        User user = userRepository.userFor(this.tenantId(), userName);
        if (!user.hasPermissionTo(Permission.Forum.StartDiscussion)) {
            throw new IllegalStateException("User may not start forum discussion.");
        }

        String authorUser = user.userName();
        String authorName = user.person().name().asFormattedName();
        String authorEmailAddress = user.person().emailAddress();

        // 创建 discussion
        Discussion discussion = new Discussion(this.tenant(), this.forumId(), 
            DomainRegistry.discussionRepository().nextIdentity(),
            authorUser, authorName, authorEmailAddress, subject);

        return discussion;
    }
}
```

以上代码展现了一种不好的设计，我们不应该在这里引用 User, 更不用说资源库了，甚至连 Permission 都不应该在此出现。这种错误的设计导致他们忽略了本应该存在的建模概念 -- Author。他们并没有把有关联的属性放在一个值对象中，而是使用一些分离的数据元素来解决问题。

现在，团队成员希望做一些临时性的改进来增强领域模型的稳定性，他们有如下选择：
- 可以将模型重构成 **职责层(Responsibility Layers)**。即将安全和权限功能下放到比当前模型更低的一个逻辑层。虽然这些层得到了正确的分离(权限校验功能被内聚在了一起)，但他们依然应该保留在模型中，因为他们都是核心域的一部分。
- 另一种方法是采用 **隔离内核(Segregated Core)**。我们可以在整个 *协作上下文* 中搜索对安全和权限相关逻辑的引用。然后将身份和访问组件分离到一个单独的包中。使用隔离内核的时机是当你有一个非常重要的限界上下文，但是其中模型的关键部分却被大量起辅助作用的功能所掩盖了。这里的安全和权限功能显然只是起辅助作用的。团队成员最终意识到，他们需要一个单独的 *身份与访问上下文* 作为 *协作上下文* 的通用子域。

公司领导层认为，将软件功能分离到新的业务服务可能孵化出一个新的 SaaS 产品，因此决定使用第二种方案。即首先采用隔离内核的方法，将安全和权限功能迁移到单独的一个模块中，更近一步在此基础上开发出新的 *身份与访问上下文*。

采用隔离内核方法重构后的代码如下:

```java
/**
 * 论坛应用服务，注意它在领域模型的上层
 */
public class ForumApplicationService {

    @Transactional
    public Discussion startDiscussion(String tenantId, String userName, String forumId, String subject) {
        Tenant tenant = new Tenant(tenantId);
        Forum forum = this.forum(tenant, forumId);
        if (forum == null) {
            throw new IllegalStateException("Forum does not exist.");
        }

        // 在应用服务中检查权限
        Author author = this.collaboratorService.authorFrom(tenant, authorId);

        Discussion newDiscussion = forum.startDiscussion(this.forumNavigationService(), author, subject);
        this.discussionRepository.add(newDiscussion);

        return newDiscussion;
    }
}

/**
 * 论坛 Forum 实体
 */
public class Forum extends Entiry {
    
    public Discussion startDiscussion(ForumNavigationService forumNavigationService,
                                      Author author, String subject) {
        if (this.isClosed()) {
            throw new IllegalStateException("Forum is closed.");
        }

        // 核心域只需要实现和协作行为相关的功能
        Discussion discussion = new Discussion(this.tenant(), this.forumId(), 
                                               forumNavigationService.nextDiscussionId(),
                                               author, subject);
        
        DomainEventPublisher.instance().publish(new DiscussionStarted(discussion.tenant(), 
                                                                      discussion.forumId(),
                                                                      discussion.discussionId(),
                                                                      discussion.subject()));
        return discussion;
    }
}
```

#### 身份与访问上下文

现在，多数企业级应用程序都需要某种形式的安全和权限组件，这样的组件用以对用户进行认证和授权。正如之前所说，一种幼稚的做法是将这样的组件嵌入到每一个离散的系统中，这将导致每一个系统都产生筒仓效应(silo effect)，即各个系统各自为政，未能建立共识。

SaaSOvation 公司决定开发一个新的限界上下文 —— *身份与访问上下文*。该产品被命名为 IdOvation。

如下图所示，*身份与访问上下文* 向多租户订阅方提供支持。更进一步，当我们关心的状态由于模型行为而发生改变时，系统将发布 **领域事件**。这些领域事件通常采用 "名词+动词" 的形式来命名，动词应该是过去分词形式，比如 tenantProvisioned, UserPasswordChanged, PersonNameChanged 等。

![]( {{site.url}}/asset/implementing-ddd-sample-id-context.jpg )

#### 敏捷管理上下文

考虑到敏捷开发的流行，SaaSOvation 公司决定启动 ProjectOvation 项目，这是一个新的核心域，CollabOvation 团队的顶尖开发人员将被调入到 ProjectOvation 项目中以传授 DDD 经验。

ProjectOvation 产品关注于敏捷项目的管理，使用 Scrum 作为迭代和增量式的项目管理框架。ProjectOvation 遵循传统的 Scrum 项目管理模型，其中包括产品、产品负责人、团队、待定项、计划发布和冲刺等。

SaaSOvation 公司希望把 CollabOvation 作为附加服务加入到 ProjectOvation 中，此时 CollabOvation 便是 ProjectOvation 的一个支撑子域。

ProjectOvation 的技术负责人一开始打算将 ProjectOvation 作为 CollabOvation 模型的扩展，所以从 CollabOvation 中新建了一个分支作为基础来开发 ProjectOvation。这种做法是一个很严重的错误。

对 ProjectOvation 的一个需求是: 通过一系列自洽的应用服务来完成业务操作。开发团队希望 ProjectOvation 对其他限界上下文的依赖程度尽可能小。即便 IdOvation 和 CollabOvation 由于种种原因不能工作了，ProjectOvation 也应该能够自行运行。

![]( {{site.url}}/asset/implementing-ddd-sample-project-context.jpg )


## 第 3 章: 上下文映射图

### 上下文映射图为什么重要

在开始采用 DDD 时，首先你应该为你当前的项目绘制一个 **上下文映射图(Context Map)**，其中应该包含你项目中当前的限界上下文和它们之间的集成关系。

![]( {{site.url}}/asset/implementing-ddd-abstract-context-map.jpg )

上下文映射图主要帮助我们从解决方案空间的角度看待问题。

当你为一个大型企业进行限界上下文之间的集成时，你可能需要与大泥球进行交互。你希望他们提供一套新的 API，但他们不打算这么做。此时你的团队和大泥球维护团队的关系便成了 **客户方-供应方** 关系。由于大泥球团队决定维持现状，你的团队不得不陷入一种 **遵奉者** 关系中。这样的关系可能导致你的项目延期交付或者彻底失败。尽早绘制上下文映射图，这样可以迫使你仔细思考你的项目和你所依赖项目之间的关系。

#### 绘制上下文映射图

上下文映射图不需要很正式，它不是企业架构，也不是系统拓扑图。但是，它可以用于高层次的架构分析，指出诸如集成瓶颈之类的架构不足。它展示了一种组织动态能力，可以帮助我们识别出有碍项目进展的一些管理问题。

#### 产品和组织关系

这里，我们简单重复一下 SaaSOvation 公司正在开发的 3 个产品。
1. CollabOvation —— 一款社交协作产品。该产品允许注册用户发布对业务有价值的内容，发布方式是一些流行的基于 Web 的工具，比如论坛、共享日历、博客和 wiki 等。开发团队后来从 CollabOvation 中提取出了 IdOvation 模型，对于 CollabOvation 来说，IdOvation 是一个 **通用子域**，而 CollabOvation 本身又作为 ProjectOvation 的 **支撑子域**。
2. IdOvation —— 一款可重用的身份和访问管理产品。IdOvation 为注册用户提供安全的、基于角色的访问管理。
3. ProjectOvation —— 一款敏捷项目管理产品。这是 SaaSOvation 公司的新核心域。

这些限界上下文之间的关系如何，不同开发团队的关系又如何？在 DDD 中，存在多种组织模式和继承模式：
- 合作关系(Partnership): 两个限界上下文的开发团队要么一起成功，要么一起失败，此时他们需要建立一种合作关系。他们需要一起协调开发计划和集成管理。
- 共享内核(Shared Kernel): 对模型和代码的共享将产生一种紧密的依赖性。我们需要为共享的部分模型指定一个显示的边界，并保持共享内核的小型化。共享内核具有特殊的状态，在没有与另一个团队协商的情况下，这种状态是不能改变的。
- 客户方-供应方开发(Customer-Supplier Development): 当两个团队处于一种上下游关系时，上游团队可能独立于下游团队完成开发，此时下游团队的开发可能会受到很大影响。因此，在上游团队的计划中，我们应该估计到下游团队的需求。
- 遵奉者(Conformist): 在存在上下游关系的两个团队中，如果上游团队已经没有动力提供下游团队之所需，下游团队便孤军无助了，只能盲目地使用上游团队的模型。
- 防腐层(Anticorruption Layer): 当共享内核、合作关系或客户方-供应方关系无法顺利实现时，对于下游客户来说，你需要根据自己的领域模型创建一个单独的层，该层作为上游系统的委派向你的系统提供功能。
- 开放主机服务(Open Host Service): 定义一种协议，让你的子系统通过该协议来访问你的服务。你需要将该协议公开，这样任何想要与你集成的人都可以使用该协议。
- 发布语言(Published Language): 在两个限界上下文之间翻译模型需要一种公用的语言。此时你应该使用一种发布出来的共享语言来完成集成交流。发布语言通常与开发主机服务一起使用。
- 另谋他路(SeparateWay): 如果两套功能没有显著的关系，那么他们是可以被完全解耦的。
- 大泥球(Big Ball of Mud): 当我们检查已有系统时，经常会发现系统中存在混杂在一起的模型，他们之间的边界是非常模糊的。此时你应该为整个系统绘制一个边界，然后将其归纳在大泥球范围之列。

在上下文映射图中，我们使用以下缩写来表示各种关系：
- ACL 表示防腐层;
- OHS 表示开放主机服务;
- PL 表示发布语言;

#### 映射 3 个示例限界上下文

最开始，SaaSOvation 已经知道应该创建一个名为 "协作上下文" 的限界上下文。他们希望创建第二个限界上下文，但不知道如何将这个上下文从核心域中分离出来：

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-1.jpg )

在对子域进行了分析，或对问题空间进行了评估之后，团队绘制的上下文映射图如下图所示。从一个限界上下文中分离出了两个子域。由于子域和限界上下文最好保持一对一的关系，我们应该将原先的协作上下文分离成两个限界上下文:

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-2.jpg )

对子域和边界的分析要求我们做出决定。当人们需要使用 CollabOvation 的功能时，他们扮演的是参与者、作者和主持者等角色。有了这样的角色分离，我们便可以绘制出一个更高层次的限界上下文，如下图所示，团队使用了分离内核对系统进行了重构：

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-3.jpg )

当下一个 ProjectOvation 项目启动时，团队将使用这个新的核心域 —— *敏捷项目管理上下文* 来增强已有的上下文映射图。如下图所示:

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-4.jpg )

#### 协作上下文

*身份与访问上下文* 通过 REST 的方式对外发布服务。作为该上下文的客户，*协作上下文* 通过传统的类似于 RPC 的方式获取外部资源。它高度依赖于远程服务，不具有自洽性。为了满足交付计划，SaaSOvation 愿意接受这种继承方式。以下是放大后的上下文映射图：

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-5.jpg )

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-6.jpg )

下游系统中的边界对象(boundary object)采用同步的方式向上游系统获取资源。当获取到远程模型数据后，边界对象取出所需数据，再将其翻译成适当的值对象实例。*身份与访问上下文* 中一个具有 Moderator 角色的 User 被翻译成了 *协作上下文* 中的 Moderator 值对象。

REST 或者 RPC 这种系统间继承方式有一个问题，他们更容易产生有损性能延迟，并且有可能导致调用彻底失败。CollabOvation 团队急切地希望解决这个问题。

#### 敏捷项目管理上下文

为了达到比 RPC 更高的自洽性，*敏捷项目管理上下文* 的团队将尽量限制对 RPC 的使用，此时他们可以选择异步请求，或者事件处理等方式。

如果系统所依赖的状态已经存在于本地，那么我们将获得更大的自洽性。有人可能认为这只是对所有的依赖对象进行缓存，但这不是 DDD 的做法。DDD 的做法是：在本地创建一些由外部模型翻译而成的领域对象，这些对象保留着本地模型所需的最小状态集。为了初始化这些对象，我们只需要有限的 RPC 调用或 REST 请求。

然而，要与远程模型保持同步，最好的方式是在远程系统中采用面向消息的通知(notification)机制。消息通知可以通过服务总线发布，也可以采用消息队列或者 REST。

#### 和身份与访问上下文集成

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-7.jpg )

从上图可以看出，对于 *身份与访问上下文* 中的领域事件，系统将以 URI 的方式向外发布事件通知。

下图给出了映射图中各个交互对象所扮演的角色：

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-8.jpg )

- MemberService 是一个 **领域服务**，它向本地模型提供 TeamOwner 和 ProductOwner 对象，同时作为基本防腐层的接口。maintainService() 方法用于周期性地检查 *身份与访问上下文* 所发出的通知，该方法由通知组件 MemberSynchronizer 周期性地调用;
- MemberService 进一步把请求委派给 IdentityAccessNotificationAdapter, 该类在领域服务和远程的开放主机服务之间扮演了适配器的角色。该适配器作为远程系统的客户端而存在。
- 一旦适配器从远程的开放主机服务获取到了数据，它将调用 MemberTranslator 的 toMember() 方法将发布语言中的媒体数据翻译成本地系统的领域对象 Member。如果该对象在本地系统已经存在，则更新该对象。MemberService 的 updateMember() 方法用于更新一个 Member 对象，此时它把更新操作委派给了自己。Member 的子类有 ProductOwner 和 TeamOwner，它们反映了本地系统中的上下文概念。

#### 与协作上下文集成

ProjectOvation 将使用 CollabOvation 所提供的附加功能，比如论坛讨论和共享日历等。ProjectOvation 用户并不直接与 CollabOvation 交互。对于某个租户来说，ProjectOvation 必须决定出哪些 CollabOvation 的功能是对该租户可用的。然后，ProjectOvation 将协调对 CollabOvation 资源的创建。

考虑以下 "创建产品" 的用例：
1. 用户提供该产品的描述信息;
2. 用户希望展开团队讨论;
3. 用户向 ProjectOvation 发出产品创建请求;
4. ProjectOvation 创建该产品，同时为其创建论坛(Forum)和讨论(Discussion);

Forum 和 Discussion 必须在 *协作上下文* 中进行创建，这与 *身份与访问上下文* 的情形不同。后者中租户是已经存在的，用户、用户群、角色等信息也已经被定义好了。这对于实现系统自洽性来说是一个潜在的障碍，因为我们依赖于 *协作上下文* 来远程地创建资源。

在使用 **领域事件(Domain Event)** 和 **事件驱动架构(Event-Driven Architecture)** 时，我们应该仔细思考最终一致性(Eventual Consitency)。在 ProjectOvation 中，当 ProductInitiated 事件产生时，该事件将由本地系统进行处理。本地系统要求 Forum 和 Discussion 在远程完成创建，这可以通过 RPC 或消息机制完成。在使用 RPC 时，如果远程的 CollabOvation 系统不可用，ProjectOvation 将定期重试直到成功为止。如果采用消息机制，ProjectOvation 将向 CollabOvation 发出消息，在资源创建成功之后，CollabOvation 同样会以消息的形式返回。当 ProjectOvation 接收到返回的消息时，它将使用新建 Discussion 的标识引用来更新本地的 Product 对象。

我们可以考虑使用 状态模式(State) 来处理资源不可用的情况:

```java
public enum DiscussionAvailablity {
    ADD_ON_NOT_ENABLED, NOT_REQUESTED, REQUESTED, READY;
}

public final class Discussion implements Serializable {
    private DiscussionAvailablity availablity;
    private DiscussionDescriptor descriptor;
    ...
}

public class Product extends Entity {
    ...
    private Discussion discussion;
    ...
}
```

如果 DiscussionAvailablity 不为 READY, 参与者可以得到此时 Discussion 的相关信息。

![]( {{site.url}}/asset/implementing-ddd-context-map-sample-9.jpg )

*敏捷项目管理上下文* 在本地有 DiscussionService 和 SchedulingService, 它们是领域服务，用于管理协作系统中的讨论和日历条目。它使用了 CollaborationAdapter 作为集成 CollabOvation 的防腐层。


## 第 4 章: 架构

DDD 的一大好处是它并不需要特定的架构。由于核心域位于限界上下文中，我们可以在整个系统中使用多种风格的架构。

对架构风格和模式的选择收到功能需求的限制，比如用例或用户故事。换句话说，在没有功能需求的情况下，我们是不能对软件质量做出评判的，亦不能做出正确的架构选择。这也说明用例驱动架构在当今的软件开发中依然适用。

本章讲到的架构风格和架构模式并不是什么 "很酷" 的工具以致于我们应该处处使用。我们应该在有需要时，在能够降低失败风险时才采用这些架构。

### 分层

分层架构模式被认为是所有架构的始祖。它支持 N 层架构系统，因此被广泛用于 Web, 企业级应用和桌面应用。在这种架构中，我们将一个应用程序或者系统分为不同的层次。

下图所示为一个典型的 DDD 系统所采用的传统分层架构，其中核心域只位于架构中的其中一层，其上为**用户界面层(User Interface)** 和**应用层(Application Layer)**，其下是**基础设施层(Infrastructure Layer)**。

![]( {{site.url}}/asset/implementing-ddd-architecture-layer.jpg )

分层架构的一个重要原则是，每层只能与位于其下方的层发生耦合。在**严格分层架构(Strict Layers Architecture)**中，某层只能与直接位于其下方的层发生耦合；而**松散分层架构(Relaxed Layers Architecture)** 则允许任意上方层与任意下方层发生耦合。由于用户界面层和应用服务通常需要与基础设施打交道，许多系统都是基于松散分层架构的。

如果用户界面使用了领域模型的对象，那么此时的领域对象仅限于数据的渲染展现。在采用这种方式时，可以使用**展现模型(Presentation Model)**对用户界面与领域对象进行解耦。

由于用户可能是人，也可能是其它的系统，有时用户界面层将采用**开放主机服务**的方式对外提供 API。

应用服务(Application Service)位于应用层中。应用服务和领域服务(Domain Service)是不同的，因此领域逻辑也不应该出现在应用服务中。应用服务可以用于控制持久化事务和安全认证，或者向其它系统发送基于事件的消息通知，另外还可以用于创建邮件以发送给用户。应用服务本身并不处理业务逻辑，但它却是领域模型的直接客户。应用服务是很轻量的，它主要用于协调对领域服务的操作，比如聚合。同时，应用服务是表达用例和用户故事的主要手段。

因此，应用服务的通常用途是：接收来自用户界面的输入参数，再通过资源库获取到聚合实例，然后执行相应的命令操作，比如:

```java
@Transactional
public void commitBacklogItemToSprint(String tenantId, String backlogItemId, String sprintId) {
    BacklogItem backlogItem = backlogItemRepository.backlogItemOfId(tenantId, backlogItemId);

    Sprint sprint = springRepository.sprintOfId(tenantId, sprintId);

    backlogItem.commitTo(sprint);
}
```

如果应用服务比上述功能复杂地多，这通常意味着领域逻辑已经渗透到应用服务中了，此时的领域模型将变成贫血模型。因此，最佳实践是将应用层做成很薄的一层。当需要创建新的聚合时，应用服务应该使用**工厂**或聚合的构造函数来实例化对象，然后采用资源库对其进行持久化。应用服务还可以调用领域服务来完成和领域相关的任务操作，但此时的操作应该是无状态的。

当领域模型用于发布**领域事件(Domain Event)**时，应用层可以将订阅方注册到任意数量的事件上，这样的好处是可以对事件进行存储和转发，同时，领域模型只需要关注自己的核心逻辑。**领域事件发布器(Domain Event Publisher)** 也可以保持轻量化，而不用依赖于消息机制的基础设施。

我们应该尽量避免领域模型对象与基础设施层发生直接耦合。这就产生了一个问题，领域层总有些接口会依赖于基础设施层，比如资源库接口的实现需要基础设施层提供的持久化机制。如果资源库接口直接实现在基础设施层，则会出现基础设施层向上调用领域层的问题。比这稍好的方式是把资源库接口的实现放在应用层。更好的办法可以参考下文中的 "依赖倒置原则"。

#### 依赖倒置原则

有一种办法可以改进分层架构 —— 依赖倒置原则(Dependency Inversion Principle, DIP)，它通过改变不同层之间的依赖关系来达到改进目的。

> 高层模块不应该依赖于低层模块，两者都应该依赖于抽象。抽象不应该依赖于细节，细节应该依赖于抽象。

根据这个定义，低层服务(比如基础设施层)应该依赖于高层组件(比如用户界面层、应用层和领域层)所提供的接口。如下图所示:

![]( {{site.url}}/asset/implementing-ddd-architecture-layer-01.jpg )

我们可以在领域层中定义资源库接口，然后在基础设施层中实现该接口:

```java
package com.saasovation.agilepm.infrastructure.persistence;

import com.saasovation.agilepm.domain.model.product.*;

public class HibernateBacklogItemRepository implements BacklogItemRepository {
    ...

    @Override
    @SuppressWarnings("unchecked")
    public Collection<BacklogItem> allBacklogItemsCommittedTo(Tenant tenant, SprintId sprintId) {
        Query query = this.session().createQuery("from -BacklogItem as _obj_ where _obj_.tenant = ? and _obj_.sprintId = ?");
        query.setParameter(0, tenant);
        query.setParameter(1, sprintId);

        return (Collection<BacklogItem>)query.list();
    }

    ...
}
```

采用依赖倒置原则，使领域层和基础设施层都只依赖于由领域模型所定义的抽象接口。由于应用层是领域层的直接客户，他将依赖于领域层接口，并且间接地访问资源库和由基础设施层提供的实现类。应用层可以使用不同的方式来获取这些实现，包括**依赖注入(Dependency Injection)**、**服务工厂(Service Factory)**和**插件(Plug In)**。

有趣的是，当我们在分层架构中采用依赖倒置原则时，我们可能会发现，事实上已经不存在分层的概念了。无论高层还是低层，它们都只依赖于抽象，好像把整个分层架构都给推平了一样。如果我们将分层架构推平，再向其中加入一些对称性将变得如何？

### 六边形架构(端口与适配器)

![]( {{site.url}}/asset/implementing-ddd-architecture-layer-02.jpg )

我们通常将客户与系统交互的地方称为"前端", 同样，我们将系统中获取、存储持久化数据和发送输出数据的地方称为 "后端"。但是，六边形架构提倡用一种新的视角来看待整个系统。如上图所示。该架构中存在两个区域，分别是 "外部区域" 和 "内部区域"。在外部区域中，不同的客户均可以提交输入；而内部的系统则用于持久化数据，并对程序输出进行存储(比如数据库)，或者在中途将输出转发到另外的地方(比如消息)。

上图中，六边形的每条不同的边代表了不同种类的端口，端口要么处理输入，要么处理输出。

我们应该根据应用程序的功能需求来创建用例，而不是客户数量说着输出机制。当应用程序通过 API 接收到请求时，他将使用领域模型来处理请求，其中便包括对业务逻辑的执行。

以下代码表示通过 JAX-RS 发布的 RESTful 资源。当请求到达 HTTP 的输入端口时，相应的适配器将对请求的处理委派给应用服务:

```java
@Path("/tenants/{tenantId}/products")
public class ProductResource extends Resource {
    private ProductService productService;

    @GET
    @Path("{productId}")
    @Produces({ "application/vnd.saasovation.projectovation+xml" })
    public Product getProduct(@PathParam("tenantId") String tenantId,
                              @PathParam("productId") String productId, 
                              @Context Request request) {
        Product product = productService.product(tenantId, productId);
        if (product == null) {
            throw new WebApplicationException(Response.Status.NOT_FOUND);
        }

        return product;
    }
}
```

JAX-RS 所提供的 Java 注解完成了适配器的大部分功能，他们负责解析资源路径，并将参数转化为 String 类型参数。

对于上图中右侧的端口和适配器，我们应该如何看待呢？我们可以将资源库的实现看作是持久化适配器，该适配器用于访问先前存储的聚合实例，或者保存新的聚合实例。

六边形架构的一大好处是，我们可以轻易地开发用于测试的适配器。整个应用程序和领域模型可以在没有客户和存储机制的条件下进行设计开发。

### 面向服务架构(Service-Oriented Architecture, SOA)

面向服务架构对于不同的人具有不同的意思，我们考虑由 Thomas 所定义的一些 SOA 原则:

| --- | --- |
| 服务契约 | 通过契约文档，服务阐述自身的目的与功能 |
| 松耦合 | 服务将依赖关系最小化 |
| 服务抽象 | 服务只发布契约，而向客户隐藏内部逻辑 |
| 服务重用性 | 一种服务可以被其它服务所重用 |
| 服务自洽性 | 服务自行控制环境与资源保持独立性，这有助于保持服务的一致性和可靠性 |
| 服务无状态性 | 服务负责消费方的状态管理，这不能与服务的自洽性发生冲突 |
| 服务可发现性 | 客户可以通过服务元数据来查找服务和理解服务 |
| 服务组合型 | 一种服务可以由其它的服务组合而成，而不管其它服务的大小和复杂性如何 |

![]( {{site.url}}/asset/implementing-ddd-architecture-layer-03.jpg )

### REST

在过去几年中，REST(Representational State Transfer)成为了一种被广泛使用，甚至被滥用的架构流行语。有些人认为在使用 REST 时我们需要将 URL 查询参数传递给方法；还有人认为 REST 就是用 HTTP 来发送 JSON 数据。以上解释都是错误的。

#### REST 作为一种架构风格

REST 是一种架构风格。架构风格之于架构，就像设计模式之于设计一样。它将不同架构实现所共有的东西抽象出来，使得我们在谈及到架构时不至于陷入技术细节中。分布式系统架构存在着多种架构风格，包括客户端-服务器架构风格和分布式对象风格。

#### RESTful 服务器的关键方面

首先，资源是关键的概念。通常每种资源都有一个 URI, 更重要的是，每个 URI 都需要指向某个资源 —— 即你向外界暴露的 "东西"。资源是具有展现(representation)和状态的，这些展现的格式可能不同。客户通过资源的展现于服务器交互，格式可以为 XML, JSON, HTML 或二进制数据。

另一个关键方面是无状态通信，此时我们将采用具有自描述功能的消息。无状态通信保证了不同请求之间的相互独立性，这在很大程度上提高了系统的可伸缩性。

如果你将资源看做对象，那么你应该问问它们应该拥有什么样的接口。这个问题的答案是 REST 的另一个关键点，他将 REST 与其它架构风格区别开来。你可以调用的方法集合是固定的。每一个对象都支持相同的接口。在 RESTful HTTP 中，对象方法便可以表示为可操作资源的 HTTP 动词，其中最重要的有 GET,PUT,POST 和 DELETE。

当我们将 HTTP 动词应用在这些资源上时，我们实际上是在调用资源的行为。比如 GET 方法只能用于 "安全" 的操作: (1) 它可能完成一些客户并没有要求的动作行为; (2) 它总是读取数据; (3) 它可能被缓存起来。

有些 HTTP 方法是幂等的，即我们可以安全地对失败的请求进行重试，这些方法包括 GET, PUT 和 DELETE 等。

最后，通过使用超媒体(Hypermedia), REST 服务器的客户端可以沿着某种路径发现应用程序可能的状态变化。对于服务器来说，这意味着在返回中包含对其它资源的链接，由此客户便可以通过这些链接访问到相应的资源。

#### RESTful 客户端的关键方面

RESTful HTTP 客户端可以通过两种方式在不同资源之间转移，一种是上文提到的超媒体，一种是服务端的重定向。

#### REST 和 DDD

RESTful HTTP 是有诱惑力的，但我们不建议将领域模型直接暴露给外界，因为这样会使系统接口变得脆弱，原因在于对领域模型的每次改变都会导致系统接口的改变。要将 DDD 与 REST 结合起来使用，我们有两种方式:

第一种是为系统接口层单独创建一个限界上下文(一个程序、一个服务之类的)，再在此上下文中通过适当的策略来访问实际的核心模型。在这种方法中，系统接口模型通常是根据领域模型来设计的，但是更好、更自然的方法应该是根据用例来设计。另外，我们还可以为这种方式自定义一种媒体类型。

另一种方法用于需要使用标准媒体类型的时候。如果某种媒体类型并不用于支持单个系统接口，而是用于一组相似的客户端-服务端交互场景，此时我们可以创建一个领域模型来处理每一种媒体类型。

### 命令和查询职责分离 —— CQRS

从资源库 Repository 中查询所有需要显示的数据是困难的，特别是需要显示来自不同聚合类型与实例的数据时。领域越复杂，这种困难就越大。

CQRS(Command-Query Responsibility Segregation) 是将紧缩(Stringent)对象(或者组件)的设计原则 和 命令-查询分离(CQS)应用在架构模式中的结果。
> 一个方法要么是执行某种动作的命令，要么是返回数据的查询，而不能两者皆是。换句话说，问题不应该对答案进行修改。更正式的解释是，一个方法只有在具有参考透明性(referentially transparent)时才能返回数据，此时该方法不会产生副作用。

在对象层面，这意味着:
- 如果一个方法修改了对象的状态，该方法便是一个命令(Command), 它不应该返回数据。在 Java 和 C# 中，这样的方法应该声明为 void.
- 如果一个方法返回了数据，该方法便是一个查询(Query), 此时它不应该通过直接或间接的手段修改对象的状态。在 Java 和 C# 中，这样的方法应该以其返回的数据类型进行声明。

现在，对于同一个模型，考虑将那些纯粹的查询功能从命令功能中分离出来。聚合将不再有查询方法，而只有命令方法。资源库也将变成只有 add() 和 save() 方法(分别支持创建和更新操作)，同时只有一个查询方法，比如 fromId()。这个唯一的查询方法将聚合的身份标识作为参数，然后返回该聚合实例。资源库不能使用其它方法来查询聚合，比如对属性进行过滤等。在将所有查询方法移除之后，我们将此时的模型称为命令模型(Command Model)。但是我们仍然需要向用户展示数据，为此我们创建第二个模型，该模型专门用于优化查询，我们称之为查询模型(Query Model)。

因此，领域模型将一分为二，命令模型和查询模型分开进行存储。最终，我们得到的组件系统如下图所示：

![]( {{site.url}}/asset/implementing-ddd-architecture-cqrs.jpg )

#### 客户端和查询处理器

上图最左侧的客户端可以是 Web 浏览器，也可以是定制开发的桌面应用程序。它们调用运行在服务端的一组查询处理器，查询处理器表示一个只知道如何向数据库执行基本查询(比如 SQL)的简单组件。

查询处理器不存在多么复杂的分层，至多是对数据库进行查询，然后将查询结果以某种格式进行序列化而已。

#### 查询模型

查询模型是一种非规范化数据模型，它并不反映领域模型，只是用于数据展示。如果数据库是 SQL 数据库，那么每张数据库表便是一种数据显示视图。表视图可以通过多张表进行创建，此时每张表代表整个显示数据的一个逻辑子集。

#### 客户端驱动命令处理

客户端向服务器发送命令(或者间接地执行应用服务)以在聚合上执行相应的行为操作，此时的聚合即属于命令模型。

#### 命令处理器

客户端提交的命令将由服务端的命令处理器(Commmand Processor)所接收。

命令处理器可以有不同的风格:
- 分类风格(categorized style), 此时多个命令处理器位于同一个应用服务中，每一个应用服务都拥有多个方法，每个方法用于处理某种类型的命令。优点是简单。
- 专属风格(dedicated style), 每种命令都对应于某个单独的类，并且该类只有一个方法。优点是处理器职责单一。

无论采用哪种风格，我们都应该在不同的处理器之间进行解耦，不能使一个处理器依赖于另一个处理器。

命令处理器通常只完成有限的功能。如果是创建功能，它会创建一个聚合示例，再将这个聚合实例添加到资源库中。如果是其它动作，那么从资源库中取到这个实例，然后调用实例中的方法即可:

```java
@Transactional
public void commitBacklogItemToSprint(String tenantId, String backlogItemId, String sprintId) {
    BacklogItem backlogItem = backlogItemRepository.backlogItemOfId(tenantId, backlogItemId);

    Sprint sprint = sprintRepository.sprintOfId(tenantId, sprintId);

    backlogItem.commitTo(sprint);
}
```

该命令处理器执行结束后，一个聚合实例将被更新，同时命令模型还将发布一个领域事件。领域事件可能导致另一些聚合实例的同步更新，最终，这些聚合实例都将于本次事务所修改的聚合实例保持最终一致性。

#### 命令模型执行业务行为

命令模型上每个方法在执行完成时都将发布**领域事件**。例如:

```java
public class BacklogItem extends ConcurrencySafeEntity {
    ...
    public void commitTo(Sprint sprint) {
        ...
        DomainEventPublisher.instance()
        .publish(new BacklogItemCommitted(this.tenent(), this.backlogItemId(), this.sprintId()));
        ...
    }
}
```

在命令模型更新之后，如果我们希望查询模型也得到相应的更新，那么从命令模型中发布的领域事件便是关键所在。

在使用事件源时，领域事件也被用于持久化修改后的聚合。

#### 事件订阅器更新查询模型

一个特殊的事件订阅器用于接收命令模型所发出的所有领域事件。有了领域事件，订阅器会根据命令模型的更改来更新查询模型。


### 事件驱动架构

事件驱动架构(Event-Driven Architecture, EDA)是一种用于处理事件的生成、发现和处理等任务的软件架构。

![]( {{site.url}}/asset/implementing-ddd-architecture-layer-02.jpg )

回顾我们之前提到的六边形架构，事件驱动架构不一定要与六边形架构一起使用。上图中的三角形表示了限界上下文所使用的消息机制。输入事件所使用的端口和其它客户端所使用的端口是不同的。输出事件也将使用另一个不同的端口。

进出六边形的事件可能有不同的类型，而我们需要特别关心的是领域事件。

我们可以引入多个六边形来表示整个企业范围内的事件架构，如下图所示。

![]( {{site.url}}/asset/implementing-ddd-architecture-eda.jpg )

#### 管道和过滤器

基于消息的系统通常呈现出一种管道和过滤器风格。

例如以下统计 303 出现次数的 shell 命令:

```bash
cat phone_numbers.txt | grep 303 | wc -l
```

1. cat 命令向标准输出流输出 txt 中的内容。
2. grep 命令从标准输出流中读取数据，即是 cat 命令的输出结果。
3. wc 命令统计读取的行数。

在以上命令工具中，每个工具都接收一个数据集，对其进行处理，再输出另一个数据集。每个工具都充当着过滤器的作用。事件驱动架构中的管道和过滤器与上面的命令行例子拥有某些相同的特征。

#### 长时处理过程(也叫 Saga)

我们可以对上面的管道和过滤器的例子进行扩展，从而得到另一种事件驱动的、分布式的并行处理模式 —— **长时处理过程(Long-Running Process)**。


