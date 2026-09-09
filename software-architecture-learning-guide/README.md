# 软件设计与架构学习指引

这份指引回答三个问题：软件架构究竟在研究什么，相关知识应按什么依赖顺序理解，以及阅读时应该抓住哪些主线。

它按知识依赖组织阅读方向，不规定完成时间、编程语言和练习项目。你可以用自己正在维护的系统理解其中概念；如果暂时没有合适项目，只阅读正文和推荐资料也可以。

## 先理解“架构”在解决什么

软件设计关注代码如何组织、对象如何协作、职责如何分配。软件架构把观察范围扩大到模块、数据、运行进程、团队和质量目标，并关心那些一旦选错就很难修改的决定。

两者没有绝对分界。一个函数的命名通常是局部设计问题；数据库边界、跨进程通信和一致性模型通常是架构问题。但如果某个看似局部的接口被几十个模块依赖，它也可能具有架构影响。

可以用四个问题判断一个决定是否值得从架构角度讨论：

- 它会影响多少代码、数据、部署单元或团队？
- 修改它需要付出多大迁移成本？
- 它保护的是哪条业务约束或质量目标？
- 有什么代码、测试、运行数据或历史经验支持这个判断？

因此，学习架构不应从记忆模式名称开始，而应从理解变化、边界和权衡开始。

## 知识之间的依赖关系

```text
可理解、可验证的代码
        ↓
行为、状态与不变量
        ↓
抽象、组合与变化方向
        ↓
模块、用例与数据边界
        ↓
运行时协作与分布式约束
        ↓
领域语言、质量属性与架构决策
```

这条顺序不是按术语难度排列，而是按理解依赖排列：不知道怎样保护一个对象的有效状态，很难判断领域边界；分不清模块边界和部署边界，也很容易把微服务理解成文件夹的远程版本。

## 第一部分：可理解、可验证的代码

### 为什么从这里开始

架构最终通过代码落地。如果一次小改动都难以定位影响范围，增加层次、接口或服务只会放大理解成本。这个阶段关注修改反馈，而不是追求某一种编程风格。

### 应理解的内容

- 命名、控制流和错误处理怎样影响阅读者的推理负担；
- 纯计算与 I/O、副作用之间的区别；
- 测试如何固定外部可观察行为，而不是冻结内部实现；
- 小步重构为什么需要快速、独立、可重复的反馈；
- 过程式、函数式和面向对象表达各自擅长呈现什么。

阅读代码时，重点观察一项需求变化需要跳转多少位置、理解多少无关细节，以及测试失败能否清楚指出被破坏的行为。不要把“函数短”“文件多”直接等同于清晰。

建议从 [Google Engineering Practices 的代码评审指南](https://google.github.io/eng-practices/review/reviewer/looking-for.html)、[Refactoring Catalog](https://refactoring.com/catalog/) 和 [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) 开始。前者提供审视代码的维度，后两者分别帮助理解结构调整和测试反馈。

当你能够从行为、依赖和修改成本解释一段代码是否清晰，而不只是引用风格规则时，可以继续理解状态与模型。

## 第二部分：行为、状态与不变量

### 为什么接着学习模型

函数可以计算某个时刻的结果，但业务对象通常会经历一段生命历程。例如订单不能从已取消重新回到处理中，任务不能在依赖未完成时开始。这些“始终必须成立”的规则称为不变量（invariant）。

### 应理解的内容

- 实体（Entity）为什么依靠身份延续，值对象（Value Object）为什么依靠值相等；
- 状态、命令和状态迁移之间的关系；
- 封装如何保护决定，而不只是隐藏字段；
- 多态的核心是行为可替换，而不是共享几个字段；
- 组合与继承分别会产生什么耦合；
- 简单数据记录、领域对象和应用服务有什么差别。

学习对象设计时，先问“哪些状态不应被创建”“哪个对象最了解这条规则”，再讨论类和接口。状态机也不是所有对象的默认形式；只有状态转换本身构成重要业务规则时，它才明显有益。

可以阅读 [Stanford CS 190 的 Modular Design](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php%3Ftopic%3DmodularDesign) 理解信息隐藏和深模块，配合 [Tell, Don’t Ask](https://martinfowler.com/bliki/TellDontAsk.html) 思考行为归属。需要严格理解子类型时，再读 Liskov 与 Wing 的 [A Behavioral Notion of Subtyping](https://dl.acm.org/doi/10.1145/197320.197383)。

当你能明确写出重要状态及其允许迁移，并能解释规则为什么属于某个对象而不是调用方时，再进入抽象与模式。

## 第三部分：抽象、组合与变化方向

### 抽象不是越多越好

抽象的价值在于隐藏一项变化，使调用者不必同时理解所有细节。它的成本是间接层、导航、调试和概念数量。没有真实变化方向的抽象，往往只是把简单代码改写成猜测未来的框架。

### 应理解的内容

- 高内聚、低耦合描述的不是目录外观，而是代码一起变化的方式；
- 依赖倒置关注高层策略不被低层技术决定牵着走；
- Strategy 适合替换算法，Adapter 适合翻译外部协议；
- Factory、Command、Observer、Decorator 等模式各自隐藏什么变化；
- SOLID、YAGNI、DRY 等原则为什么会在不同情境下互相拉扯；
- 一个接口怎样做到表面简单，同时真正隐藏大量复杂性。

学习设计模式时，不要按目录逐个背诵。先理解模式所面对的问题、相互冲突的作用力（forces）、协作结构和代价，再去看代码形式。同一个结构在缺少对应变化时可能只是过度设计。

[Design Patterns](https://www.pearson.com/en-us/subject-catalog/p/design-patterns-elements-of-reusable-object-oriented-software/P200000009480) 适合理解模式的原始问题描述，[Refactoring.Guru](https://refactoring.guru/design-patterns) 适合作为结构索引，[YAGNI](https://martinfowler.com/bliki/Yagni.html) 则提醒提前抽象本身也有成本。

当你能用“哪种变化会发生、谁应该不受影响、为此付出什么代价”解释一个抽象时，就可以把视角扩大到模块与系统结构。

## 第四部分：模块、用例与数据边界

### 从对象协作走向系统组织

对象设计解决局部规则和协作；模块设计决定哪些知识应该放在一起、哪些决定应被隐藏。模块是逻辑边界，不必是独立部署的服务。

### 应理解的内容

- 按业务能力和可能变化的设计决定划分模块；
- 应用用例如何协调一次用户意图，领域模型如何保护规则；
- 端口与适配器如何隔离数据库、Web、消息和供应商协议；
- DTO、领域对象、数据库记录和映射各自承担什么职责；
- Repository、事务脚本、领域模型和 Service Layer 的适用范围；
- 逻辑模块、运行进程和部署单元为什么不能混为一谈；
- 模块化单体为什么常常比过早拆服务更容易验证边界。

理解数据边界时，重点不是给每张表创建一个 Repository，而是判断哪些规则必须在同一一致性范围内完成。数据库事务也无法撤销已经发送的邮件、执行的支付或其他外部副作用。

建议阅读 Parnas 的 [On the Criteria To Be Used in Decomposing Systems into Modules](https://dl.acm.org/doi/10.1145/361598.361623)，再看 [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) 和 [Patterns of Enterprise Application Architecture Catalog](https://martinfowler.com/eaaCatalog/)。前者解释为什么按可能变化的决定分解，后两者帮助连接应用核心、外部适配器和企业数据模式。

当你能画出实际依赖方向、说明事务边界，并解释为什么某个模块当前不需要独立部署时，再学习运行时架构。

## 第五部分：运行时协作与分布式约束

### 先区分三类结构

- **代码组织**：分层、模块、插件核心；
- **运行时协作**：进程内调用、请求响应、工作队列、发布订阅、事件流；
- **部署拓扑**：单体、多个进程、函数或独立服务。

三者可以独立组合。采用消息并不意味着必须使用微服务；拆成多个进程也不意味着已经建立清晰模块。

### 应理解的内容

- 同步调用和异步消息对延迟、故障与调试的不同影响；
- 超时、重试、幂等、重复投递、乱序和背压；
- 数据所有权、事务边界与最终一致性；
- 分层、客户端—服务端、微内核、SOA、微服务和 Serverless 的主要约束；
- CQRS、Event Sourcing 和事件驱动为什么是可以分别采用的决定；
- 可观察性为何是分布式协作的一部分，而不是事后增加的日志。

学习任何架构风格时，都要同时问它改善了什么质量、引入了什么新的失败方式，以及团队是否有能力承担其运维成本。没有独立扩缩容、故障隔离、技术边界或团队自治需求时，分布式拆分通常没有充分依据。

可以先比较 [Azure Architecture Styles](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/)，再按问题查阅 [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/)；需要理解服务拆分权衡时阅读 [Microservices](https://martinfowler.com/articles/microservices.html)。CQRS 与 Event Sourcing 应分别阅读 [CQRS](https://martinfowler.com/bliki/CQRS.html) 和 [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)。

当你能描述一次请求或消息经过哪些边界、每一步怎样失败、系统如何恢复并让用户看到真实状态时，再进入架构评审与演进。

## 第六部分：领域语言、质量属性与架构决策

### 架构好坏取决于上下文

相同结构在不同业务中可能得到相反评价。低延迟交易、内部报表和个人笔记应用对一致性、可靠性、性能、成本与可修改性的优先级完全不同。架构判断必须连接业务语义和质量目标。

### 应理解的内容

- 统一语言（Ubiquitous Language）如何减少业务与代码之间的翻译损耗；
- 边界上下文（Bounded Context）如何允许同一个词在不同范围拥有不同模型；
- 聚合如何表达一致性边界，而不是把相关对象全部装进一个容器；
- 性能、可靠性、安全、可修改性、可观察性和成本等质量属性；
- 质量属性场景如何把“高性能”“高可用”变成可讨论、可验证的条件；
- ADR 如何记录背景、候选方案、权衡、结论和重审信号；
- 上下文图、组件图、时序图和部署图分别服务什么沟通目的；
- 测试、指标、故障复盘和业务变化如何推动架构演进。

DDD 不是微服务的前置仪式，也不要求所有系统采用复杂领域模型。它最有价值的部分是帮助团队发现语言冲突、业务边界和真正需要一致性的规则。

可以用 [Domain-Driven Design Reference](https://www.domainlanguage.com/ddd/reference/) 建立术语入口，用 [SEI Quality Attribute Workshops](https://www.sei.cmu.edu/library/quality-attribute-workshops/) 学习质量场景，再通过 [C4 Model](https://c4model.com/) 选择合适的表达层级。可靠性与运行目标可继续阅读 [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)。

到这里，判断架构时应能形成一条完整因果链：

```text
业务变化或质量目标
        ↓
需要保护的规则与边界
        ↓
候选结构及其代价
        ↓
当前决定
        ↓
测试、图、数据或运行事实
        ↓
未来重新评估的条件
```

## 经典书籍地图

经典书籍的价值不在于年代或名气，而在于它们为一类反复出现的问题建立了稳定语言。下面按知识依赖分组；同组之间可以互相对照，不需要按书单顺序全部读完。

### 代码与模块设计

- [《Code Complete, 2nd Edition》](https://www.informit.com/store/code-complete-9780735619678)，Steve McConnell。适合补齐软件构造的整体视野，包括复杂度、控制流、防御式编程、质量与协作。内容很广，优先查阅正在遇到的构造问题，不必作为架构书逐章通读。
- [《Refactoring, 2nd Edition》](https://martinfowler.com/books/refactoring.html)，Martin Fowler。用于理解怎样通过小步、保持行为的修改持续改善既有设计。先读开篇示例、重构原则、坏味道与测试相关章节，再把目录当作日常参考。
- [《A Philosophy of Software Design, 2nd Edition》](https://web.stanford.edu/~ouster/cgi-bin/aposd.php)，John Ousterhout。重点关注复杂度的来源、信息隐藏、深模块、通用与专用接口。适合与 Clean Code、SOLID 等规则对照阅读，避免把任何一套建议绝对化。
- [《Design Patterns: Elements of Reusable Object-Oriented Software》](https://www.pearson.com/en-us/subject-catalog/p/design-patterns-elements-of-reusable-object-oriented-software/P200000009480)，Erich Gamma、Richard Helm、Ralph Johnson、John Vlissides。先读模式如何描述问题、作用力和协作，再按当前设计问题查询具体模式；不要把 23 个模式当作必须使用的组件目录。

### 应用、数据与领域边界

- [《Patterns of Enterprise Application Architecture》](https://martinfowler.com/books/eaa.html)，Martin Fowler 等。连接领域逻辑、应用层、关系数据库、Web 表现和并发问题。先读前半部分的组织思路，再按需查询 Transaction Script、Domain Model、Data Mapper、Repository 等模式。
- [《Domain-Driven Design: Tackling Complexity in the Heart of Software》](https://www.domainlanguage.com/ddd/)，Eric Evans。适合业务规则复杂、语言容易分裂的系统。先理解知识消化、统一语言和模型驱动设计，再进入聚合、领域事件与边界上下文；简单 CRUD 系统不必完整套用。
- [《Enterprise Integration Patterns》](https://www.enterpriseintegrationpatterns.com/)，Gregor Hohpe、Bobby Woolf。为异步消息、路由、转换、端点和错误处理提供共同词汇。它更适合作为模式语言和参考手册，在系统真正跨进程或跨应用协作时阅读。

### 架构、生产运行与演进

- [《Software Architecture in Practice, 4th Edition》](https://www.sei.cmu.edu/library/software-architecture-in-practice-fourth-edition/)，Len Bass、Paul Clements、Rick Kazman。可作为架构主干书，重点理解质量属性、架构策略、分析、演进以及业务和组织环境怎样影响架构。
- [《Documenting Software Architectures: Views and Beyond, 2nd Edition》](https://www.sei.cmu.edu/library/documenting-software-architectures-views-and-beyond-second-edition/)，Paul Clements 等。适合需要向不同利益相关者表达架构时查阅，重点是怎样选择有用视图、记录跨视图信息和设计理由，而不是学习更多画图符号。
- [《Release It!, 2nd Edition》](https://store.pragprog.com/titles/mnee2/release-it-second-edition/)，Michael Nygard。通过生产故障理解稳定性模式、反模式、隔离、超时、恢复和运维现实。准备学习分布式协作或负责线上系统时阅读最有价值。
- [《Designing Data-Intensive Applications》](https://dataintensive.net/)，Martin Kleppmann。系统比较存储、复制、分区、事务、批处理与流处理背后的模型和权衡。它适合在已有模块与数据边界概念之后阅读，不适合作为软件设计的第一本书。

如果只选三本作为主线，可以依次阅读《A Philosophy of Software Design》《Patterns of Enterprise Application Architecture》和《Software Architecture in Practice》。当工作重心转向生产可靠性或数据系统时，再分别进入《Release It!》或《Designing Data-Intensive Applications》。

## 怎样使用这份指引

第一次阅读时只看每部分的“为什么”和“应理解的内容”，先建立知识地图。第二次再根据自己最不清楚的部分进入推荐资料。遇到具体工程问题时，从问题所在层级向前回看依赖，不必每次从头学习。

外部资料是解释来源，不是完成度清单。能否用自己的语言说明概念的本质、边界和权衡，比读过多少书更重要。

主线之外还可以继续深入分布式系统、安全、性能与容量、组织架构或特定领域建模。是否进入这些方向，应由实际问题决定，而不是因为架构知识看起来必须“全部学完”。

## 内容来源与公开边界

这份学习指引使用自己的结构、措辞和解释，不复刻第三方路线图的节点、描述或编排。最初选题受到 [roadmap.sh](https://roadmap.sh/) 启发，但本文不是其镜像、翻译或官方版本，也没有隶属、授权或背书关系。第三方内容只保留原始页面链接；详细约定见仓库根目录的 [公开发布规范](../PUBLICATION.md)。
