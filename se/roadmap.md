## 软件架构阅读路线

### 1. System Design Primer
**目标：建立系统设计知识地图**

掌握：

- 负载均衡
- 缓存
- 数据库复制
- 分片
- 消息队列
- 一致性
- CDN
- 限流

---

### 2. The Architecture of Open Source Applications
**目标：理解真实开源项目如何组织架构**

关注：

- 模块划分
- 依赖关系
- 数据流
- 架构决策

---

### 3. Fundamentals of Software Architecture
**目标：建立完整的软件架构基础**

掌握：

- 架构特征
- 模块化
- 耦合与内聚
- 分层架构
- 事件驱动架构
- 微服务架构
- 架构权衡
- 架构决策

---

### 4. Distributed Systems — van Steen & Tanenbaum
**目标：补齐分布式系统理论基础**

掌握：

- RPC
- 时间与时钟
- 协调
- Leader Election
- Consensus
- Replication
- Consistency
- Fault Tolerance

---

### 5. Designing Data-Intensive Applications
**目标：深入理解数据密集型系统**

重点：

- 数据模型
- 存储引擎
- 编码
- 复制
- 分片
- 事务
- 分布式系统问题
- 一致性与共识
- 批处理
- 流处理

---

### 6. Google SRE
**目标：理解生产系统的可靠性设计**

掌握：

- SLI / SLO / SLA
- Error Budget
- Monitoring
- Alerting
- Capacity Planning
- Incident Response
- Automation
- Reliability

---

### 7. DDD Quickly + DDD Reference
**目标：学习业务建模与系统边界设计**

掌握：

- Domain
- Subdomain
- Bounded Context
- Entity
- Value Object
- Aggregate
- Domain Event
- Ubiquitous Language

---

### 8. Software Architecture: The Hard Parts
**目标：训练复杂架构决策能力**

重点：

- 服务拆分
- 数据所有权
- 分布式事务
- 同步与异步
- 架构 Trade-off
- 架构决策记录

---

### 9. Patterns of Distributed Systems
**目标：理解常见分布式系统实现模式**

掌握：

- WAL
- Leader/Follower
- Quorum
- Lease
- Version Vector
- Generation Clock
- Replication Pattern

---

### 10. MIT 6.5840 Distributed Systems
**目标：通过实现深入理解分布式系统**

重点完成：

- MapReduce
- Raft
- KV Raft
- Sharded KV

---

### 11. 开源源码阅读
**目标：把理论映射到真实系统**

推荐顺序：

```text
Redis
↓
etcd
↓
Kafka
↓
TiKV / TiDB
↓
Kubernetes
```

---

## 总路线

```text
System Design Primer
        ↓
AOSA
        ↓
Fundamentals of Software Architecture
        ↓
Distributed Systems
        ↓
DDIA
        ↓
Google SRE
        ↓
DDD
        ↓
Software Architecture: The Hard Parts
        ↓
Patterns of Distributed Systems
        ↓
MIT 6.5840
        ↓
源码阅读
```

### 最终阅读目标

```text
建立架构知识地图
→ 理解分布式与数据系统
→ 掌握可靠性设计
→ 学会业务边界划分
→ 学会架构权衡与决策
→ 能阅读并分析真实大型系统
```