# Agent Mission Control：软件设计与架构实践课程

这是一套面向工程实践的原创课程。它不复刻任何第三方路线图的节点、描述、版式或编排，也不作为第三方内容的镜像。

最初的选题灵感来自 [roadmap.sh](https://roadmap.sh/)。本文与 roadmap.sh 没有隶属、授权或背书关系；如果需要查看其最新内容，请直接访问其网站。

## 课程目标

学完后，你不只是“认识一些模式”，而是能够：

- 用快速测试和清晰代码建立可靠的修改反馈；
- 用对象、值和状态机表达业务规则与不变量；
- 根据变化方向选择抽象、组合与设计模式；
- 划分高内聚、低耦合、可独立理解的模块；
- 区分代码组织、运行时协作和部署边界；
- 用领域语言、质量属性、图和 ADR 解释架构决定；
- 识别过度设计，并知道何时删除抽象或退回简单方案。

## 能力递进

    快速反馈
      → 行为与状态模型
        → 可复用的设计语言
          → 模块和数据边界
            → 分布式协作
              → 领域驱动的架构判断

这里的顺序按能力依赖组织，而不是按术语目录组织。后一阶段会反复使用前一阶段的产物；每次扩展都要能说明新复杂度解决了什么真实问题。

## 贯穿项目：Agent Mission Control

用你最熟悉的语言实现一个本地 AI Agent 任务编排与可观测平台：

- 用户创建由任务与依赖组成的工作流；
- 调度器挑选已经满足条件的任务；
- 执行器运行 Agent 或本地工具；
- 系统记录等待、运行、成功、失败、重试和取消；
- 时间线展示任务事件与失败原因；
- 插件机制允许替换执行器、存储和通知方式。

项目按课程逐步生长：

1. 纯内存、确定性的任务选择核心；
2. 能保护不变量的工作流领域模型；
3. 可替换的调度、执行和工具插件；
4. 带持久化适配器的模块化单体；
5. 异步 Worker、消息与可观察的事件时间线；
6. 可选接入真实 LLM，并完成一次架构评审。

真实模型放到最后接入。这样，调度与状态错误不会和网络、费用、限流或模型随机性混在一起。

## 学习方法

每个阶段都执行同一闭环：

1. 预习：先写下“它解决什么问题”和当前猜想。
2. 输入：完成必读，再按实践中的障碍选择核心与拓展资料。
3. 微练习：先在一个函数、对象或小图上验证概念。
4. 项目实践：改造 Agent Mission Control，并用测试、图或 ADR 留证。
5. 复盘：闭卷说明收益、代价、适用边界和一个不该使用的场景。

资料标签：

- 必读：进入项目实践前完成；
- 核心：只精读与当前任务相关的部分；
- 拓展：遇到对应问题时再读，不阻塞主线。

每个主题使用下面的记录结构：

    本质：它解决什么问题？
    边界：它不解决什么问题？
    权衡：得到什么，又付出什么？
    证据：测试、代码、图或 ADR 在哪里？

## 课程地图

| 阶段 | 核心问题 | 项目里程碑 |
| --- | --- | --- |
| 1 | 怎样安全、清晰地修改代码？ | 可重复测试的调度核心 |
| 2 | 怎样让模型保护有效状态？ | 工作流领域模型 |
| 3 | 怎样以最小机制容纳变化？ | 可替换策略与插件 |
| 4 | 怎样划分模块、用例和数据边界？ | 模块化单体 |
| 5 | 怎样选择运行时协作方式？ | 异步 Worker 与事件时间线 |
| 6 | 怎样以业务和质量证据做架构决策？ | 架构评审与可选真实 LLM |

---

## 阶段 1：先建立快速反馈

### 本阶段回答

怎样让下一次修改既容易理解，又能快速确认没有破坏行为？

### 知识结构

1. 可读性：意图明确的命名、一致的错误处理、减少嵌套。
2. 控制流：顺序、分支、循环、早返回，以及复杂度从何而来。
3. 数据与副作用：纯计算、不可变输入、I/O 边界。
4. 小步重构：先固定行为，再改变结构。
5. 测试反馈：快速、独立、可重复，并能清楚定位失败。
6. 三种表达视角：过程式流程、函数组合、对象协作。

### 补充学习资料

按下面顺序阅读，不需要一次读完整本书：

1. 必读：[Google Engineering Practices — What to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html)。聚焦 Design、Complexity、Tests、Naming 和 Comments，把“干净”变成可检查的问题。
2. 必读：[Martin Fowler — Refactoring Catalog](https://refactoring.com/catalog/)。先练 Extract Function、Decompose Conditional 和 Introduce Parameter Object。
3. 必读：[The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)。理解为什么快速测试是重构的安全网。
4. 核心：[Dijkstra — A Case against the GO TO Statement](https://www.cs.utexas.edu/~EWD/transcriptions/EWD02xx/EWD215.html)。理解程序结构为什么要便于人推理，而不只是记住“不要用 goto”。
5. 核心：[Python Functional Programming HOWTO](https://docs.python.org/3/howto/functional.html)。即使不用 Python，也可以学习纯函数、迭代器、组合与副作用。
6. 拓展：[SICP JavaScript](https://sicp.sourceacademy.org/sicpjs.pdf)。选读第 1 章的过程抽象和第 3 章的状态。

### 阅读到实践的桥

先写一个故意直接、但行为正确的 selectReadyTasks。随后只做三类变化：

- 把依赖判断提取成可命名的规则；
- 把解析、日志等 I/O 移出纯计算；
- 每次重构前先写一个能失败的行为测试。

再分别用显式流程、函数组合和策略对象表达同一规则。比较的不是代码行数，而是增加“优先级”和“失败分支不阻塞无关任务”时，要改多少地方、引入多少状态。

### 项目实践

- 覆盖空工作流、未完成依赖、失败依赖、非法状态；
- 让核心计算不依赖 HTTP、数据库或框架对象；
- 保存一次重构前后 diff；
- 写一段复盘：哪种表达最适合当前规则，为什么另外两种暂不采用。

### 完成标准

- [ ] 测试能独立、乱序、重复运行。
- [ ] 核心规则可以只靠输入和输出解释。
- [ ] 能指出一处降低认知复杂度的重构证据。
- [ ] 能举例说明“拆得更小”为什么不一定更清晰。

### 进入下一阶段前

如果 selectReadyTasks 已经稳定，但调用者仍能随意拼出非法任务状态，说明问题已经从“函数是否清楚”转向“模型能否保护规则”，这正是阶段 2 的入口。

---

## 阶段 2：建立行为与状态模型

### 本阶段回答

怎样让非法状态难以创建，让业务规则靠模型本身成立？

### 知识结构

1. 实体与值对象：身份变化和按值比较的区别。
2. 不变量：任何公开操作完成后都必须成立的规则。
3. 封装与抽象：保护状态，只暴露调用者需要的能力。
4. 多态与可替换：依赖契约，而不是依赖类型判断。
5. 组合与继承：默认组合，只有满足真实“是一个”关系和行为契约时才继承。
6. 变化原则：按变化原因拆分，让高层策略不依赖低层细节。
7. 简单性原则：不要为未经证实的未来需求预建扩展点。

### 补充学习资料

1. 必读：[Oracle — Object-Oriented Programming Concepts](https://docs.oracle.com/javase/tutorial/java/concepts/index.html)。把 Object、Class、Inheritance、Interface 翻译成你所用语言的机制。
2. 必读：[Stanford CS 190 — Modular Design](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php%3Ftopic%3DmodularDesign)。学习接口与实现、信息隐藏、深模块和浅模块。
3. 核心：[Robert C. Martin — The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)。把每条原则放回“什么变化会迫使代码一起改”的问题中。
4. 核心：[Martin Fowler — YAGNI](https://martinfowler.com/bliki/Yagni.html)、[Command Query Separation](https://martinfowler.com/bliki/CommandQuerySeparation.html) 与 [Tell, Don’t Ask](https://martinfowler.com/bliki/TellDontAsk.html)。对照理解简单性、行为归属和副作用边界。
5. 拓展：[Liskov 与 Wing — A Behavioral Notion of Subtyping](https://dl.acm.org/doi/10.1145/197320.197383)。先读摘要和结论，需要严谨讨论子类型时再深读。
6. 拓展：[A Philosophy of Software Design 第二版节选](https://web.stanford.edu/~ouster/cgi-bin/aposd2ndEdExtract.pdf)。用它校正“所有类都必须很小”等机械规则。

### 阅读到实践的桥

先写出状态迁移表，再写类：

| 当前状态 | 命令 | 允许结果 |
| --- | --- | --- |
| waiting | start | running |
| running | succeed | succeeded |
| running | fail | failed 或 waiting-for-retry |
| waiting/running | cancel | cancelled |

为每个不允许的组合写测试。如果测试只能通过绕过公开 API、直接改字段构造非法状态，模型还没有真正封装规则。

### 项目实践

- 实现 Workflow、Task、Run、TaskId 和 RetryPolicy；
- 禁止缺失依赖、自依赖和循环依赖；
- 终态任务不能重新进入运行态；
- RetryPolicy 始终保证次数和退避参数有效；
- 实现 FakeAgentExecutor 与 LocalScriptExecutor 的共同契约；
- 写一个继承破坏可替换性的反例，再改成组合。

### 完成标准

- [ ] 非法状态无法通过公开 API 创建。
- [ ] 业务对象不依赖 Web、ORM 或消息框架。
- [ ] 能解释实体、值对象、服务和数据记录的区别。
- [ ] 每个接口都有真实使用者，而不是为了“解耦”预先创建。

### 进入下一阶段前

当模型有效但新增调度算法、执行器或工具会反复修改条件分支时，你已经获得采用模式的证据。不要在证据出现之前进入阶段 3。

---

## 阶段 3：把模式当作设计语言

### 本阶段回答

怎样用小而明确的协作结构容纳变化，而不是堆叠模式名称？

### 知识结构

每个模式都用四个问题学习：

1. 问题：哪种变化正在重复伤害代码？
2. 作用力：哪些要求彼此冲突？
3. 结构：对象或模块如何协作？
4. 代价与退出：多了哪些间接层？何时应该删除它？

优先掌握这些与项目直接相关的模式：

- Strategy：替换调度或重试算法；
- Adapter：隔离外部模型、存储或工具协议；
- Factory：集中不可避免的构造差异；
- State：让迁移规则显式化；
- Command：表示可记录、可重试的意图；
- Observer：分发状态变化；
- Decorator：叠加日志、指标、重试等横切能力；
- Microkernel 思路：稳定核心与可插拔能力分离。

### 补充学习资料

1. 必读：[Microsoft — Strategy pattern](https://learn.microsoft.com/en-us/shows/visual-studio-toolbox/design-patterns-strategy)。先抓住可替换算法与调用方稳定性。
2. 核心：[Refactoring.Guru — Design Patterns](https://refactoring.guru/design-patterns)。把它作为结构图和示例索引，不要顺序背诵。
3. 核心：[Design Patterns: Elements of Reusable Object-Oriented Software](https://www.pearson.com/en-us/subject-catalog/p/design-patterns-elements-of-reusable-object-oriented-software/P200000009480)。只查当前要用的模式原文。
4. 核心：[Patterns of Enterprise Application Architecture — Catalog](https://martinfowler.com/eaaCatalog/)。先观察模式如何描述问题、结构和权衡。
5. 拓展：[Pattern-Oriented Software Architecture](https://www.wiley.com/en-us/Pattern-Oriented+Software+Architecture%2C+Volume+1%2C+A+System+of+Patterns-p-9780471958697)。当需要把对象模式提升到系统结构时再读。

### 阅读到实践的桥

为每个准备引入的模式写一张“模式卡”：

    触发变化：
    现有痛点：
    最小结构：
    新增成本：
    删除条件：

没有具体触发变化的模式不得进入项目。先用条件分支实现一次，再通过第二个真实实现证明抽象值得存在。

### 项目实践

- 用 Strategy 支持 FIFO 和优先级调度；
- 用 Adapter 隔离 FakeAgent 与未来的真实模型；
- 用 Command 表示 StartTask、RetryTask 和 CancelTask；
- 用事件订阅更新 Timeline；
- 为日志与指标选择显式包装或 Decorator；
- 主动删除至少一个没有真实收益的模式，并记录理由。

### 完成标准

- [ ] 每个模式都对应一个已经出现的变化。
- [ ] 测试能替换实现而不改变领域规则。
- [ ] 没有以字符串类型判断代替多态边界。
- [ ] 能说清模式带来的间接层和调试成本。

### 进入下一阶段前

单个模式只能改善局部协作。当多个功能仍围绕同一数据表和框架目录互相穿透时，需要讨论模块、用例与持久化边界。

---

## 阶段 4：划清模块、用例与数据边界

### 本阶段回答

怎样让系统按业务能力而不是按框架文件夹生长？

### 知识结构

1. 高内聚与低耦合：经常一起变化的东西放在一起。
2. 信息隐藏：模块隐藏可能变化的决定，而不只是隐藏字段。
3. 依赖方向：核心策略不认识传输、存储与供应商细节。
4. 逻辑边界与部署边界：模块化不等于微服务。
5. 用例层：协调一次用户意图，不吞掉领域规则。
6. 数据边界：DTO、Mapper、Repository、ORM 和事务各自负责什么。
7. 企业应用组织：简单事务脚本与领域模型的适用条件。

### 补充学习资料

1. 必读：[David Parnas — On the Criteria To Be Used in Decomposing Systems into Modules](https://dl.acm.org/doi/10.1145/361598.361623)。理解按“可能变化的设计决定”分解模块。
2. 必读：[Stanford CS 190 — Modular Design](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter18/lecture.php%3Ftopic%3DmodularDesign)。这次重点复习深模块、接口复杂度和信息泄漏。
3. 核心：[Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)。学习应用核心、端口与适配器的边界观。
4. 核心：[Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)。按需阅读 Transaction Script、Domain Model、Service Layer、Data Mapper、Repository、Identity Map。
5. 核心：[Microsoft — Design the infrastructure persistence layer](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)。忽略语言细节，关注 Repository、ORM 与领域层的关系。
6. 拓展：[SEI — Software Architecture](https://www.sei.cmu.edu/our-work/software-architecture/)。建立架构质量与分析的后续入口。

### 阅读到实践的桥

画一张当前依赖图，用箭头表示“需要知道对方”。逐条检查：

- 领域模块是否知道 HTTP 或 ORM 类型？
- 用例是否只是转发，还是承担了事务协调？
- 一个数据模型是否同时充当 API、数据库和领域模型？
- 模块边界是否能用测试替换外部实现？

先保持单进程部署，只重构逻辑边界。没有独立扩缩容、故障隔离或团队自治证据时，不拆服务。

### 项目实践

把项目整理成模块化单体：

- Workflow：工作流定义、依赖和状态规则；
- Execution：任务执行与重试；
- Tooling：工具注册与调用；
- Timeline：事件查询和展示；
- Application：命令、查询和事务协调；
- Adapters：CLI/API、数据库、文件和外部服务。

同时完成：

- 内存 Repository 与一个持久化 Adapter；
- 领域对象与数据库/API DTO 的显式映射；
- 至少一条边界测试，阻止适配器依赖倒流；
- 一张组件图，标出所有跨模块契约。

### 完成标准

- [ ] 领域逻辑可以脱离数据库运行。
- [ ] 模块名使用业务语言，而不是框架层名。
- [ ] 依赖图无循环，跨边界调用有明确契约。
- [ ] 能解释为什么当前选择模块化单体。

### 进入下一阶段前

模块边界稳定后，再研究跨进程协作。否则消息队列只会把混乱的调用关系变成更难观察的异步关系。

---

## 阶段 5：设计运行时协作

### 本阶段回答

什么时候同步调用已经不够，消息、事件或独立服务的代价才值得？

### 知识结构

先区分三类决定：

- 代码组织：分层、模块、插件核心；
- 运行时协作：请求响应、工作队列、发布订阅、事件流；
- 部署拓扑：单进程、多个进程、函数或独立服务。

再比较常见结构及代价：

- 分层与客户端—服务端；
- 模块化单体与微内核；
- 面向服务与微服务；
- Serverless；
- CQRS；
- Event Sourcing。

CQRS、Event Sourcing、事件驱动和微服务不是一组必须捆绑采用的技术。每项都要有独立证据。

### 补充学习资料

1. 必读：[Microsoft Azure Architecture Styles](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/)。先比较每种风格的约束、收益和适用场景。
2. 核心：[Microsoft — N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)。用它区分逻辑层和物理层。
3. 核心：[Enterprise Integration Patterns — Messaging Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/)。按需阅读 Message Channel、Competing Consumers、Publish-Subscribe、Idempotent Receiver。
4. 核心：[Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)。重点阅读服务边界、独立部署和去中心化数据带来的权衡。
5. 核心：[Martin Fowler — CQRS](https://martinfowler.com/bliki/CQRS.html) 与 [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)。分别判断，不默认一起使用。
6. 拓展：[AWS Prescriptive Guidance — Event sourcing pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/event-sourcing.html)。关注重放、最终一致性和运维复杂度。

### 阅读到实践的桥

先写失败模型，再选通信方式：

| 风险 | 需要回答 |
| --- | --- |
| 重复投递 | 命令如何幂等？ |
| 消息丢失 | 谁负责持久化与重试？ |
| 乱序 | 状态机如何拒绝过期事件？ |
| Worker 崩溃 | 任务如何重新可见？ |
| 网络分区 | 用户看到什么中间状态？ |
| 追踪困难 | 如何关联一次工作流的日志和事件？ |

如果同步调用已经满足延迟、吞吐和故障要求，就保留同步方案。

### 项目实践

- 把执行从应用进程移到异步 Worker；
- 定义带唯一标识、版本和关联标识的消息；
- 实现幂等消费、重试上限和失败队列；
- 用事件更新 Timeline 读模型；
- 为重复、乱序和 Worker 中断编写测试；
- 写 ADR 比较“模块化单体 + Worker”与“拆成多个服务”。

### 完成标准

- [ ] 能画出一个任务从命令到完成事件的时序图。
- [ ] 能解释至少一次投递下的幂等策略。
- [ ] 可观察状态能区分排队、运行、失败和等待重试。
- [ ] 服务边界来自运行与组织证据，而不是流行度。

### 进入下一阶段前

系统结构已经能运行，但架构是否“好”仍取决于业务语言和质量目标。阶段 6 将把这些隐含判断变成可评审证据。

---

## 阶段 6：用领域与质量证据做架构判断

### 本阶段回答

怎样证明架构服务于业务和质量目标，并让决定可演进？

### 知识结构

1. 领域语言：代码、文档和讨论使用同一组业务词汇。
2. 边界上下文：同一个词在不同业务边界可以有不同模型。
3. 聚合与一致性：哪些规则必须在一个事务边界内成立。
4. 领域事件与服务：表达已经发生的事实和不属于单一实体的规则。
5. 质量属性：性能、可靠性、安全、可修改性、可观察性与成本。
6. 决策记录：背景、候选方案、权衡、结论和触发重审的条件。
7. 架构表达：系统上下文图、容器/组件图、时序图和运行视图。
8. 演进机制：测试、遥测、容量数据和故障复盘推动调整。

### 补充学习资料

1. 必读：[Domain-Driven Design Reference](https://www.domainlanguage.com/ddd/reference/)。先建立 Entity、Value Object、Aggregate、Domain Event 和 Bounded Context 的共同词汇。
2. 必读：[Martin Fowler — Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)。用它检查领域行为是否被搬空到服务脚本。
3. 核心：[Microsoft — Design a microservice domain model](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)。只取领域建模方法，不把“微服务”当作前提。
4. 核心：[EventStorming](https://www.eventstorming.com/)。用事件发现流程、冲突语言和边界，而不是把便签结果直接当最终设计。
5. 核心：[SEI — Quality Attribute Workshops](https://www.sei.cmu.edu/library/quality-attribute-workshops/)。把“要高性能、要可靠”改写成可讨论的场景。
6. 核心：[C4 model](https://c4model.com/)。只画对沟通有用的层级，并保持图与代码一致。
7. 拓展：[Google SRE Workbook](https://sre.google/workbook/table-of-contents/)。按需阅读 SLO、监控和事件响应章节。

### 阅读到实践的桥

先写三条可验证的质量场景，例如：

    在单个 Worker 崩溃时，正在处理的任务应在指定时间内重新进入可执行状态，
    且不会产生重复的外部副作用。

再把每项架构决定连接到一个场景。无法连接的复杂结构应被质疑。

### 项目实践

- 维护领域词汇表并修正含糊命名；
- 明确 Workflow、Execution、Tooling 与 Timeline 的模型边界；
- 定义聚合不变量和事务边界；
- 完成系统上下文图、组件图和关键时序图；
- 为队列、持久化与模型适配器分别写 ADR；
- 增加结构化日志、关联标识、关键指标和失败诊断信息；
- 可选：通过现有 Adapter 接入真实 LLM，核心领域不得依赖供应商 SDK。

### 完成标准

- [ ] 每个核心模块都能用一句业务语言说明责任。
- [ ] 每个关键架构决定都连接到业务约束或质量场景。
- [ ] ADR 同时记录没有选择的方案和重审条件。
- [ ] 图、代码与运行证据没有明显矛盾。
- [ ] 能指出至少一个应保持简单而未采用的高级方案。

---

## 毕业评审

不要以“资料看完”作为结课。安排一次 45 分钟评审，提交：

1. 一个可运行的 Agent Mission Control 演示；
2. 领域模型、不变量和状态迁移说明；
3. 组件图与一条关键任务时序图；
4. 三份最重要的 ADR；
5. 重复消息、乱序事件和 Worker 中断的测试证据；
6. 一次“删除不必要复杂度”的记录；
7. 下一阶段真实需求出现时，架构将从哪里演进。

评审问题：

- 哪个边界最可能先变化？
- 哪项复杂度有实测证据，哪项只是猜测？
- 如果负载下降十倍，哪些组件可以删除？
- 如果团队增加到三个小组，当前边界是否支持独立交付？
- 如果更换数据库、消息系统或 LLM 供应商，核心业务会改多少？

## 主线之外的补充模块

这些内容很重要，但不应打断当前六阶段主线。遇到真实需求时再进入。

### 补充 A：分布式系统基础

与阶段 5 相连。学习超时、重试、幂等、背压、时钟、复制、一致性和共识。推荐从 [Designing Data-Intensive Applications](https://dataintensive.net/) 和 [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.824/) 开始。

### 补充 B：安全设计

横跨阶段 4–6。学习威胁建模、最小权限、秘密管理、审计日志和供应链安全。入口：[OWASP Threat Modeling](https://owasp.org/www-community/Threat_Modeling)。

### 补充 C：性能与容量

与阶段 5、6 相连。先测量延迟分布、吞吐、资源利用率和队列长度，再决定缓存、并行或扩容。入口：[Google SRE Book](https://sre.google/sre-book/table-of-contents/)。

### 补充 D：源码阅读

与阶段 3、4 相连。挑选一个成熟框架，追踪一次请求、一次事务或一次插件加载；用调用图和边界图解释设计，不以“读完仓库”为目标。

## 维护原则

- 课程结构由本仓库独立维护，不追求与任何第三方页面逐项同步。
- 新增主题前先说明它填补了哪一个能力缺口。
- 外部链接只作为延伸阅读；不在仓库中保存未经授权的副本、截图或导出文件。
- 外部材料失效时，优先替换为作者或机构的一手公开来源。
- 每次项目演进都应保留测试、图或 ADR 中至少一种可复核证据。
