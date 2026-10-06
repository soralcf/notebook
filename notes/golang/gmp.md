# Go 调度模型：GMP

> 源码基线 **Go 1.27.1**，路径 `/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/`。GMP 的实现细节跨版本变化明显，读到更早的资料先对齐版本号。

## 一句话

GMP 是 Go runtime 的用户态调度器：把几十万个 goroutine 分摊到少量 OS 线程上跑，同时保证「同时真正在执行 Go 代码」的数量不超过 `GOMAXPROCS`。

## 打个比方

一家工厂：

- **P = 工位**，全厂只有固定几个（`GOMAXPROCS` 个）
- **M = 工人**，必须**占到一个工位**才能干活
- **G = 待办的活**，几十万件，堆在工位旁边的筐里
- 每个工位**自带一套常用工具**（`mcache`），不用跑去公共工具间排队
- 工人去仓库领料（系统调用）时，要**先把工位让出来**

GMP 的全部设计，都在回答三个问题：工位给谁、活儿怎么分、工人卡住时工位怎么办。

## 主干

P 有三重身份。记住这三个**钩子词**，就记住了整个模型。

| 钩子词 | 一句话 | 直接后果 |
|---|---|---|
| **许可证** | 没有 P 的 M 不能执行 Go 代码 | 并发度 = P 数，与线程数解耦 |
| **资源包** | 每个 P 自带 `mcache` | 小对象分配不抢全局锁 |
| **稀缺品** | P 只有 `GOMAXPROCS` 个 | G 一跑不动，runtime 就想办法把 P 让出去 |

后面所有内容，都是这三条的下游。

## 展开

### 三个角色

| | G | M | P |
|---|---|---|---|
| 是什么 | 用户态协程 | OS 线程 | 逻辑处理器 |
| 谁调度 | Go runtime | OS | Go runtime |
| 数量 | 动态，可达百万 | 动态，受阻塞影响 | `GOMAXPROCS`，默认核数 |
| 有栈吗 | 有（初始 2KB，可增长） | 有 `g0` 栈 | 没有 |
| 关键字段 | `g.sched`、`g.atomicstatus` | `m.g0`、`m.curg`、`m.p` | `runq`、`runnext`、`mcache` |

调度循环 `schedule()` → `findRunnable()` → `execute()` 永远跑在 M 的 `g0` 栈上，不在 goroutine 栈上。

### ① 许可证：为什么需要 P 这一层

只有 G 和 M 的话，goroutine 一多就得开出同样多的线程，OS 扛不住。P 把「有多少活儿想跑」和「允许多少活儿同时真跑」分开了。

- 数量关系：G 任意多 → M 按需增减 → **P 固定为 `GOMAXPROCS`**
- 所以 `GOMAXPROCS` **不是**线程数上限，它只是「同时执行 Go 代码」的上限
- M 因阻塞变多时不会占用额外的 P —— P 会被 `handoffp` 交给别的 M（见「系统调用与阻塞」）

**`GOMAXPROCS` 从 Go 1.25 起会自动变**：

- 默认跟随 cgroup 的 CPU 限额（`GODEBUG=containermaxprocs=0` 关掉容器感知）
- 周期性重新读取（`GODEBUG=updatemaxprocs=0` 关掉周期更新）
- 每次调整走一次 STW（`stwGOMAXPROCS`）
- **但用户一旦自己调过 `runtime.GOMAXPROCS`，`sched.customGOMAXPROCS` 置位，自动调整直接放弃** —— 手动设过就不再自动变

### ② 资源包：P 顺手解决了锁

每个 P 独占一份 `mcache`（小对象缓存，`runtime2.go:782`）。分配小对象时先掏自己口袋，不碰全局锁。

这是 Go 分配器能扩展到多核的关键，也是 P 这一层不能省的原因之一 —— 它不只是个计数器。

### ③ 稀缺品：runtime 大部分代码都在服务这一条

`schedule()` / `findRunnable()` 里的逻辑，本质都是「别让 P 闲着」：

| P 要闲下来的原因 | runtime 的对策 | 见 |
|---|---|---|
| 本地队列空了 | 去偷别的 P 的活儿 | 工作窃取 |
| 一个 G 占着 P 跑太久 | 抢占它，换别人上 | 抢占 |
| G 进了系统调用 | 把 P 交给别的 M | 系统调用与阻塞 |

### 工作窃取

**为什么要分层**：全局队列是唯一入口，会锁竞争。所以每个 P 配一个无锁的本地队列，本地空了再去偷 —— 用低竞争换负载均衡。

| | 本地队列 | 全局队列 |
|---|---|---|
| 数据结构 | `runq [256]` 环形 + `runnext` | 带锁的 `sched.runq` 链表 |
| 访问方式 | 无锁（属主读写，他人原子窃取） | 必须持 `sched.lock` |
| 容量 | 256 + 1 | 无上限 |
| 什么时候写 | 常规入队 | 本地队满时批量转移 |

**入队** `runqput(pp, gp, next)`（`proc.go:7529`）：

- `next=true` 写 `runnext`，否则进本地队尾
- **本地队满不是溢出单个元素，而是批量倒**：`runqputslow`（`proc.go:7575`）把队里一半（128 个）加上新来的那个，共 **129 个**一起送进全局队列

**出队** `runqget(pp)`（`proc.go:7649`）：先看 `runnext`，再看本地队头。

**`runnext` 是什么**：容量 1 的优先槽。当前 G 唤醒的 G 放进 `runnext` 后，**继承当前 G 剩余的时间片**（`runtime2.go:808-816`）。目的是让「通信后立刻阻塞」的 goroutine 成组被调度，省掉一次调度延迟。

它**可以被别的 P 偷走**：只有属主 P 能把它 CAS 成有效 G，其他 P 可以把它 CAS 成 0（`runtime2.go:818-819`、`proc.go:7720-7756`）。

`runqput` 的注释点破了它和抢占的关系：*A runnext goroutine shares the same time slice as the current goroutine... To prevent a ping-pong pair of goroutines from starving all others, we depend on sysmon to preempt "long-running goroutines".* —— 也就是说，**runnext 链的公平性完全交给 sysmon 的抢占兜底**。

**公平性兜底**：每 **61** 次调度（`pp.schedtick%61 == 0`）强制看一眼全局队列（`proc.go:3458`）。没有它，两个互相唤醒的 goroutine 可以让本地队列永远有活儿，全局队列饿死。

**偷窃怎么做**（`stealWork`，`proc.go`）：

- 最多试 **4 轮**（`stealTries = 4`）
- 起点随机（`stealOrder.start(cheaprand())`），不是从 0 号 P 顺序轮询
- 跳过 idle 的 P（`idlepMask`）
- **偷 `runnext` 和别人的 timer 只在最后一轮才试** —— 源码注释写明 *stealing from the other P's runnext should be the last resort*
- 偷的数量是 `n - n/2`，即**一半，奇数时向上取整**（`runqgrab`）
- 偷 `runnext` 之前有个 **3μs 退避**：如果目标 P 正在 `_Prunning` 且它的 curg 不在 syscall 里，先睡 3μs 再偷。注释解释：sync channel 收发约 50ns，3μs 留了约 50 倍余量，避免 G 在两个 P 之间来回抖动

**`findRunnable` 的完整查找顺序**：

| # | 来源 |
|---|---|
| 1 | `gcwaiting` → `gcstopm()` |
| 2 | safe point fn |
| 3 | timer heap（`pp.timers.check`，只取时间，不返回 G） |
| 4 | trace reader |
| 5 | GC worker |
| 6 | **公平性**：`schedtick%61==0` 且全局队列非空 → 全局队列 |
| 7 | finalizer G |
| 8 | cleanup G |
| 9 | `cgo_yield` |
| 10 | **本地队列** |
| 11 | **全局队列**：批取，上限 128（本地队列容量的一半） |
| 12 | netpoll 非阻塞探测（同一时刻只允许一个线程 poll） |
| 13 | **窃取别的 P** |
| 14 | idle GC mark |
| 15 | 阻塞式 netpoll（按最近的 timer 定超时） |
| 16 | `stopm()` |

注意第 11 条：从全局队列取是**批取**，实际数量是 `min(128, 队列长度, 队列长度/GOMAXPROCS+1)`（`proc.go:7351`）——**128 只是上限**；而第 6 步的公平性检查只取 1 个（`globrunqget()`，`proc.go:7332`）。

### 抢占

**解决什么问题**：一个 goroutine 死循环，若不能被打断，同 P 上其他 goroutine 一起饿死。

| | 协作式（≤ Go 1.13） | 异步抢占（Go 1.14+） |
|---|---|---|
| 触发者 | 运行中的 G 自己走到检查点 | `sysmon` 发信号 |
| 抢占点 | 函数序言（`morestack` 的栈增长检查） | 任意能精确还原寄存器和栈的位置 |
| `for {}` 死循环 | 抢不动 | 可被抢 |
| 开销 | 极低（一次比较） | 信号 + 栈扫描，较贵 |

Go 1.14 release notes 原文：*Goroutines are now asynchronously preemptible.* 平台例外：`windows/arm`、`darwin/arm`、`js/wasm`、`plan9/*`。

**怎么做的**：

- 信号是 `SIGURG`：`const sigPreempt = _SIGURG`（`signal_unix.go:74`）。选它的理由：Linux 上默认忽略、out-of-band 数据基本没人用、不会和用户代码的信号处理冲突
- 发起者是 **`sysmon` 线程**，周期性醒来调 `retake(now)`
- 判定条件：`pd.schedwhen + forcePreemptNS <= now`，其中 **`forcePreemptNS = 10ms`**（`proc.go:6679`、`proc.go:6715`）
- 抢占后 G 进 `_Gpreempted`，等某个 `suspendG` 把它 CAS 成 `_Gwaiting` 并负责重新入队

**`sysmon` 的睡眠是自适应的**（`proc.go:6537-6557`）：

- `idle == 0` 时从 **20μs** 起
- `idle > 50`（约 1ms）后每轮翻倍
- 上限 **10ms**
- `idle` 是「连续多少轮没能唤醒别人」的计数

它有三项职责：从 syscall 夺回 P、抢占长时间运行的 G、所有 P 都忙时兜底网络轮询。

**`retake` 用 `schedtick` 判断是不是同一时间片**（`proc.go:6708-6715`）：没变说明还在同一片；变了就重置 `schedwhen`。源码注释特意点出一个坑：同一时间片里跑了太久，可能是**一个长 G**，也可能是 **`runnext` 串起来的一串 G** 共用了同一个 `schedtick`。

**syscall 里的 G 抢不动**：`preemptone` 对正在 syscall 的 P 无效（G 和 M 都不在 Go 代码里，收不了信号），所以 `retake` 改为直接**夺走这个 P**（`sysretake`，`proc.go:6716-6721`）。

**10ms 是发起阈值，不是硬上限**：收到 `SIGURG` 后，runtime 要等到一个能精确还原状态的指令位置才能真停下。cgo、汇编、不可抢占区域都会推迟。`GODEBUG=asyncpreemptoff=1` 可以关掉异步抢占。

### 系统调用与阻塞

**为什么需要**：G 阻塞时不能拖垮整个 P，否则一个慢 IO 就废掉 `GOMAXPROCS` 分之一。

**三条路径，处理完全不同**：

| | runtime 内阻塞 | 阻塞式 syscall | 网络 IO |
|---|---|---|---|
| 例子 | channel、`sync.Mutex`、`time.Sleep` | 慢设备 `read(2)`、cgo、直接 `syscall.Syscall` | TCP read/write |
| G 状态 | `_Gwaiting` | `_Gsyscall` | `_Gwaiting` |
| 占 M 吗 | 否 | **是** | 否 |
| P 怎么办 | 留在原 P | `handoffp` 交出去 | 留在原 P |
| 谁唤醒 | runtime（`ready()`） | syscall 返回 | netpoller |

**进入 / 退出 syscall**：

- `entersyscall()` 把 G 从 `_Grunning` 置为 `_Gsyscall`（`proc.go:4776`）
- 此时 G **挂着 P 但不再拥有它**。源码注释明确写：**执行在 `_Gsyscall` 状态的代码不得访问 `g.m.p`**，因为 P 随时可能被交出去
- 阻塞路径上调 `handoffp(releasep())` 把 P 交出去（`proc.go:2014`、`proc.go:2080`、`proc.go:4869`）
- `exitsyscall()`（`proc.go:4922`）尝试重新拿回一个 P；拿不到就把 G 放回队列并 park 当前 M

**`handoffp(pp)` 的分支顺序**（`proc.go:3146`）—— 决定这个 P 接下来去哪：

| 顺序 | 条件 | 动作 |
|---|---|---|
| 1 | 本地队列或全局队列有活儿 | 起一个**非自旋** M |
| 2 | 有 trace 活儿 | 起一个非自旋 M |
| 3 | 有 GC blacken 活儿 | 起一个非自旋 M |
| 4 | `nmspinning + npidle == 0`（没人在等） | 起一个**自旋** M |
| 5 | `gcwaiting` | P 进 `_Pgcstop` |
| 6 | 有 safe point fn | 就地执行 |
| 7 | 再查一次全局队列 | 起一个非自旋 M |
| 8 | 最后一个 P 且没人在 poll 网络 | 起一个非自旋 M |
| 9 | 以上都不成立 | `pidleput` 放回 idle 列表，必要时 `wakeNetPoller` |

**自旋 M 是什么**：已经没活儿但还没 park、还在到处找活儿的线程。保留它是为了压低「来了新活儿才唤醒线程」的延迟。

启动规则很保守：

- `wakep()` 只要发现**已经有自旋 M 就直接返回**（注释：*only start one if none exist already*）
- `findRunnable` 里转自旋的条件是 `2*nmspinning < gomaxprocs-npidle`，即**自旋 M 不超过忙 P 数的一半**（注释：*Limit the number of spinning Ms to half the number of busy Ps*）。不这样限制的话，`GOMAXPROCS` 很大而实际并行度很低时会白烧 CPU

**netpoller 才是 Go 高并发网络的原因**：goroutine 阻塞在网络操作上时进 `_Gwaiting`、不占 M，IO 就绪后由 netpoller 唤醒。少量线程就能撑住大量连接 —— 比「goroutine 很轻量」这个流行说法更准确。

注意它**不覆盖所有 IO**：Linux 上普通文件 IO 仍是阻塞式 syscall。

### 状态机

**G 的一生**（按经历顺序记，别背枚举；`runtime2.go:79-119`）：

| 阶段 | 状态 | 要点 |
|---|---|---|
| 刚分配 | `_Gidle` | 未初始化 |
| 可运行 | `_Grunnable` | 在运行队列上，**不持有栈** |
| 在跑 | `_Grunning` | 持有栈，绑定 M |
| 进 syscall | `_Gsyscall` | 持有栈、绑定 M，**挂着 P 但不拥有它** |
| 阻塞在 runtime | `_Gwaiting` | channel / 锁 / timer；**有人负责叫醒它** |
| 被抢占 | `_Gpreempted` | 被 `suspendG` 停下后**还没人负责叫醒它** |
| 栈在搬 | `_Gcopystack` | 短暂过渡 |
| 结束 | `_Gdead` | 在 free list 上，可能没有栈 |
| 泄漏被抓 | `_Gleaked` | GC 栈扫描时发现（`mgc.go:1300`） |

最容易混的一对：**`_Gwaiting` 是「有人负责叫醒我」，`_Gpreempted` 是「还没人负责」**。

**P 的状态只有 5 个**（`runtime2.go:122-163`）：

| 状态 | 含义 |
|---|---|
| `_Pidle` | 空闲，在 idle 列表上，运行队列为空 |
| `_Prunning` | 被某个 M 拥有，正在跑 |
| `_Psyscall_unused` | **已废弃**（见「反直觉」） |
| `_Pgcstop` | 为 STW 停机 |
| `_Pdead` | 不再使用（`GOMAXPROCS` 缩小） |

**栈**：goroutine 栈可增长，初始 **2048 字节**（`stack.go:78` 的 `stackMin = 2048`），上限 `maxstacksize` = 64 位 **1GB** / 32 位 **250MB**，且 `maxstackceiling = 2 * maxstacksize`（`proc.go:164-172`）。

## 反直觉

- **你以为 P 就是一个 CPU 核，其实它只是 runtime 的记账单位。** 默认等于核数，但能设成任意值，也能大于核数。
- **你以为 `GOMAXPROCS` 是线程数上限，其实它只限制「同时执行 Go 代码」的数量。** 阻塞式 syscall 一多，线程数会远超它。
- **你以为 goroutine 阻塞都不占线程，其实只有 runtime 内阻塞和网络 IO 成立。** 阻塞式 syscall 会真的占住一个 M。
- **你以为 10ms 是调度器的轮转配额，其实它只是抢占的触发阈值。** Go 没有 OS 那种固定时间片轮转，主动阻塞或让出是不计时的；`sysmon` 自己还以 20μs~10ms 自适应睡眠，最坏情况下实际延迟明显大于 10ms。
- **你以为进了 `runnext` 就一定会下一个跑，其实别的 P 可能在轮到它之前就把它 CAS 走了。**
- **你以为全局队列只是本地队满时的溢出区，其实它还是公平性兜底通道。** 61 次一次的检查专门防止两个 goroutine 靠互相唤醒永久霸占本地队列。
- **你以为 P 有一个 `_Psyscall` 状态，其实它已改名 `_Psyscall_unused`。** 老资料（基于 Go 1.13 / 1.14 源码）还在讲 `_Psyscall`；Go 1.27.1 里 P「是否在系统调用中」改为**看 G 的状态**判断（`runtime2.go:143-146`）。
- **你以为被抢占的 G 就此结束，其实它只是离开运行状态，之后会被重新放回队列。**
- **你以为异步抢占取代了协作式抢占，其实函数序言的栈检查仍在**，它是热路径上极廉价的抢占点；异步抢占是补充而非替代。
- **你以为 goroutine 调度跟信号无关，其实 Go 1.14 起 Unix 程序收到的信号变多了。** 用 `syscall` 包的程序会看到更多 `EINTR` 失败 —— 这是 Go 1.14 release notes 明确提醒的副作用。
- **你以为 `GOMAXPROCS` 总会自动跟随容器限额，其实手动设置过一次就永久放弃自动调整。** 内部 `sched.customGOMAXPROCS` 置位后，自动调整逻辑直接放弃。
- **你以为被抢占的是 P，其实被抢占的是 P 上正在跑的那个 G**；P 本身是被「夺走」或「移交」。

## 和邻居的区别

**GMP vs OS 原生线程**

| | GMP | OS 线程 |
|---|---|---|
| 调度单位 | goroutine | 线程 |
| 切换成本 | 用户态，不进内核 | 系统调用 + 上下文切换 |
| 栈大小 | 初始 2KB，按需增长 | 通常固定 MB 级 |
| 抢占方式 | 协作 + `SIGURG` 异步（10ms 阈值） | 硬件时钟中断 |
| 谁能执行 | 必须持有 P | 内核决定 |

**goroutine vs async/await（有栈 vs 无栈）**

| | goroutine | async/await |
|---|---|---|
| 栈 | 有，可增长 | 无，是编译器生成的状态机 |
| 挂起点 | 任意函数调用（runtime 决定） | 必须显式写 `await` |
| 传染性 | 无 | 有（异步函数只能被异步函数调） |
| 切换开销 | 保存 / 恢复寄存器 | 编译期生成的状态跳转 |
