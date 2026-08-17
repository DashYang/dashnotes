# 通用后端与基础设施

> 核验日期：2026-07-16
>
> 本文关注可迁移的工程原理。具体 API、客户端默认值和云服务参数会演进，落地前以对应版本官方文档为准。

## 主题简介：What / Why / How

- **What**：面向高并发读、可靠持久化、异步消息和链上交易的一组通用工程模式。
- **Why**：进程崩溃、网络分区、消息重复、链重组和外部系统超时是常态；只优化“成功路径”会在恢复时破坏一致性。
- **How**：用清晰状态机、WAL/事务保证本地原子性，用幂等和 Outbox/Inbox 跨越边界，用可重放 workflow 管理长事务，再以对账和可观测性闭环。

贯穿全文的原则是：**每一个副作用都要有稳定身份、可判断状态、有限重试和事实来源。**

## 1. 并发读优化

### 1.1 RCU：Read / Copy / Publish / Reclaim

Read-Copy-Update（RCU）适合读远多于写、读路径极度敏感的共享结构：

1. Reader 读取当前指针，尽量不阻塞 writer。
2. Writer 复制旧版本，在副本上修改。
3. Writer 用原子 publish 替换全局指针。
4. 等待旧 reader 离开 grace period 后，才 reclaim 旧版本。

困难不在 copy，而在**何时确认旧对象无人引用**。常见方案包括 epoch/quiescent-state tracking、hazard pointer、reference counting。选择方案时考虑 reader 是否会阻塞、线程是否可注册、对象创建频率和内存上限。

### 1.2 内存序与回收

publish 需要 release 语义，reader 获取指针需要 acquire 或满足对应语言内存模型的同步规则，确保 reader 看到完整初始化的新对象。即使指针替换原子，字段初始化没有 happens-before 仍会读到未完成状态。

错误回收比数据竞争更隐蔽：旧 reader 仍持有裸指针时释放对象会 use-after-free。GC 语言降低内存释放风险，但“读到多个版本拼接状态”的逻辑一致性问题仍存在。

### 1.3 RCU 与读写锁

| 维度 | RCU / copy-on-write | RWMutex |
| --- | --- | --- |
| 读路径 | 极轻，常无锁 | 需要 lock/unlock，可能竞争 |
| 写成本 | 复制、发布、延迟回收 | 原地修改，写锁排他 |
| 一致性视图 | 单个版本快照 | 锁保护区间内一致 |
| 适用数据 | 配置、路由表、只读索引 | 写频繁或对象很大、复杂原地更新 |
| 主要风险 | 回收、内存峰值、旧版本滞留 | writer starvation、锁抖动、死锁 |

不要把 RCU 当作“无锁所以总更快”。写频繁、大对象复制或 reader 长时间不退出时，它会放大 CPU 和内存。

### 1.4 bRPC `DoublyBufferedData`

`DoublyBufferedData<T>` 用两份数据服务高频读：reader 通过线程局部 Wrapper 选择当前前台副本；writer 修改后台副本，翻转前台，再修改旧前台，使两份数据最终一致。

`Modify` 的回调可能执行两次，必须满足：

- 对两个相同初始状态执行得到相同结果；
- 不在回调内发送网络请求、写数据库等非幂等副作用；
- 修改结果不能依赖调用次数或随机数；
- reader 不持有 Wrapper 跨越 bthread 切换或不受支持的生命周期。

双缓冲不提供跨两个不同 DBD 对象的原子快照。需要多字段一致时，应把字段放入同一个不可变 `T` 后整体 publish。

### 1.5 高频问答

**问：RCU 为什么读快？**

答题骨架：reader 只获取已发布版本；writer copy/publish；回收延迟到 grace period。随后指出内存序和 reclamation 才是正确性核心。

**问：DBD 的 Modify 为什么执行两次？**

答题骨架：先改后台、翻转，再把相同修改应用到原前台，确保两个副本一致；因此 callback 必须确定、无外部副作用。

## 2. WAL 与持久化

### 2.1 WAL 的不变量

Write-Ahead Log 的核心规则是：**对应数据页落盘前，描述其修改的日志必须先持久化**。崩溃后从 checkpoint/LSN 开始重放日志，恢复一致状态。

- LSN 给日志全序或分段位置，用于判断哪些更新已持久化。
- redo 重做已提交但数据页未落盘的修改。
- undo 回滚未提交修改；是否需要 undo 取决于存储设计（如 steal/no-steal、MVCC）。
- checkpoint 缩短恢复扫描范围，但不能替代日志 durability。

### 2.2 fsync、Group Commit 与一致性

写入 page cache 不等于掉电持久化；系统要明确 `fsync/fdatasync` 边界。每事务一次 fsync 延迟高，group commit 将多个事务共享一次 flush，用少量等待换吞吐。

crash consistency 要考虑 torn write、write reorder、目录项和文件 rename 的持久化。数据库已经提供事务时，不要在应用层另造一套“写两个文件就算原子”的协议。

### 2.3 常见用途与边界

- 数据库用 WAL 恢复已提交事务。
- Raft 先持久化 log entry，再根据复制/commit 规则应用状态机。
- 区块节点保存 block/state journal，应对进程崩溃或短重组。
- Transactional Outbox 把“业务状态 + 待发送事件”写入同一个数据库事务。

WAL 只能约束拥有该日志的系统。它不自动让数据库更新和外部 HTTP/MQ/链上交易成为一个原子事务。

## 3. Kafka

### 3.1 Topic、Partition、Offset、Consumer Group

Kafka topic 被切成 partition；每个 partition 内记录按 offset 有序。consumer group 将 partition 分配给组内消费者，因此同一 partition 在稳定分配期通常由一个 group member 处理。

顺序保证是**分区内**的。若业务需要同一账户/订单顺序，应使用稳定 key 路由到同一 partition；全局顺序通常意味着单 partition，也就牺牲扩展性。

### 3.2 Rebalance 与消费进度

消费者加入、退出或 partition 变化会触发 rebalance。处理时间过长、心跳超时或频繁扩缩容会造成分区迁移和重复消费。应控制 poll/处理边界，必要时 pause partition、批量提交或把长任务交给可追踪的工作流。

自动提交 offset 可能在业务处理前确认，导致丢失；业务完成后才提交会在“处理完成、提交失败”窗口重复。正确目标不是祈祷窗口消失，而是让 handler 幂等。

### 3.3 交付语义

| 语义 | 典型做法 | 失败表现 |
| --- | --- | --- |
| At-most-once | 先提交进度，再处理 | 不重复，但可能丢 |
| At-least-once | 处理成功后提交 | 不丢，但可能重复 |
| Effectively-once | at-least-once + 业务幂等/唯一约束 | 传输可重复，业务效果一次 |

Kafka producer idempotence 防止同一 producer session 的重试产生重复记录。Kafka transaction 可原子写多个 Kafka partition，并把 consumed offsets 一并提交；配合 `read_committed` 支持 Kafka 内 read-process-write 的 exactly-once。它不自动覆盖外部数据库、支付网关或区块链。

### 3.4 背压、毒消息与 Schema 演进

- **背压**：以 consumer lag、处理延迟和下游容量调节并发，不能无限拉取堆在内存。
- **毒消息**：区分永久数据错误和暂时依赖失败；有限重试后进入 DLQ，并保留原 payload、错误、版本和 correlation ID。
- **重放**：handler 必须可幂等；重放前明确时间范围、目标版本和副作用开关。
- **Schema**：优先 backward/forward compatible 变更；消费者先兼容新旧，再升级生产者。
- **热点 key**：单 key 顺序与扩展冲突，必要时按实体子维度拆分并在业务层汇总。

## 4. 分布式事务与可靠消息

### 4.1 幂等键与唯一约束

客户端生成稳定 `idempotency_key`，服务端以“调用方 + 操作类型 + key”建立唯一约束并保存请求 hash、状态和响应。相同 key 不同 payload 必须拒绝，避免把两笔业务误合并。

幂等表写入和业务更新应处于同一数据库事务。只在 Redis `SETNX` 后操作数据库，Redis 过期或崩溃会重新打开重复窗口。

### 4.2 Inbox / Outbox

Inbox / Outbox 解决的不是“让 DB 和 MQ 共享一个本地事务”，而是把一个无法原子完成的跨系统操作，拆成两个可恢复的本地事务：生产侧用 Outbox 保证事件最终可发送，消费侧用 Inbox 保证重复事件只产生一次业务效果。

#### 4.2.1 问题从哪里来：DB 与 MQ 之间没有原子提交

假设下单接口既要在数据库创建订单，又要向 Kafka 发布 `OrderCreated`。这是两个独立副作用：

- `COMMIT orders` 由数据库决定成功或失败。
- `publish OrderCreated` 由 Kafka 决定成功或失败。

只调整调用顺序无法消除中间故障窗口：

1. **先写 DB，再发 MQ**：订单提交成功后进程崩溃，消息尚未发送；数据库有订单，下游库存/积分服务永远不知道。
2. **先发 MQ，再写 DB**：消息发送成功后数据库事务失败；消费者已经处理了一个实际不存在的订单。
3. **失败后重试也不充分**：producer 发送成功但等待 ack 超时，此时结果是 unknown outcome。若直接生成新消息重发，Kafka 中可能出现两个业务上相同的事件。

根因是 DB 和 MQ 没有共同的原子提交点。应用无法用一段普通代码保证“二者都成功，或者二者都失败”。

#### 4.2.2 Outbox：把“业务更新 + 发送意图”放进同一个 DB 事务

Outbox 是业务数据库中的一张待发送事件表。以创建订单为例，服务只在请求事务里做两件事：

```sql
BEGIN;

INSERT INTO orders(id, status, amount)
VALUES ('O1001', 'CREATED', 99);

INSERT INTO outbox(
    event_id, aggregate_type, aggregate_id, event_type, payload, created_at
) VALUES (
    'E9001', 'Order', 'O1001', 'OrderCreated', '{...}', CURRENT_TIMESTAMP
);

COMMIT;
```

因为两条 `INSERT` 属于同一个数据库事务，所以只会出现两种结果：

- 事务回滚：订单和事件都不存在，无需发消息。
- 事务提交：订单和待发送事件同时存在；即使服务立刻崩溃，事件仍留在 Outbox，恢复后可以继续发送。

独立 publisher 再把 Outbox 搬到 Kafka，常见方式有两种：

- **轮询发布**：查询待处理记录，用 `FOR UPDATE SKIP LOCKED`、状态抢占或 lease 避免多个 worker 同时处理，然后发送 Kafka，成功后标记 `published_at`。要处理 lease 过期、失败重试、死信和历史清理。
- **CDC 发布**：Debezium 等组件读取数据库 WAL/binlog 中已经提交的 Outbox `INSERT`，转换后发送 Kafka。CDC 省去业务轮询，但仍要监控 connector offset、lag、schema 和重放行为。

关键是 `event_id` 在第一次创建事件时就固定，publisher 重试必须复用同一个 ID，不能每次重试生成新 ID。`aggregate_id` 通常作为 Kafka key，使同一订单的事件进入同一 partition，保留实体内顺序。

#### 4.2.3 为什么 Outbox 仍可能重复发送

publisher 无法把“Kafka publish”和“Outbox 标记已发布”放进同一个普通 DB 事务。考虑这条时序：

1. publisher 把 `E9001` 成功发送到 Kafka。
2. publisher 还没来得及更新 `outbox.published_at` 就崩溃。
3. 重启后它看到 `E9001` 仍未发布，于是再次发送。

若反过来先标记再发送，又会在标记后崩溃而永久漏发。因此可靠实现通常选择 **至少一次发送（at-least-once）**：宁可重复，不可静默丢失；重复由消费侧处理。

#### 4.2.4 Inbox：把“去重记录 + 消费副作用”放进同一个 DB 事务

每个消费者在自己的数据库维护 Inbox，唯一键通常是 `(consumer_name, event_id)`。库存服务处理 `E9001` 时执行：

```sql
BEGIN;

WITH accepted AS (
    INSERT INTO inbox(consumer_name, event_id, received_at)
    VALUES ('inventory-service', 'E9001', CURRENT_TIMESTAMP)
    ON CONFLICT DO NOTHING
    RETURNING event_id
)
UPDATE inventory
SET reserved = reserved + 1
WHERE sku = 'SKU-1'
  AND EXISTS (SELECT 1 FROM accepted);

COMMIT;
```

上面的 PostgreSQL 示例用 `RETURNING` 保证只有 Inbox 真正插入一行时才更新库存；应用代码也可以检查 Inbox `INSERT` 的 affected rows 后再执行更新：

- 插入成功：第一次处理，在同一事务内更新库存。
- 唯一键冲突：该消费者已经处理过 `E9001`，跳过库存更新并返回成功。

Kafka offset 应在业务事务成功后再确认。若数据库提交后、offset 提交前进程崩溃，Kafka 会重新投递；但 Inbox 唯一约束会识别同一个 `event_id`，库存不会再次增加。若数据库事务提交前崩溃，Inbox 和库存更新一起回滚，重新投递后可以正常处理。

Inbox 的去重范围要包含消费者身份，因为同一个事件被库存服务和积分服务各处理一次是合法的；它只能保护与 Inbox 位于同一事务资源中的数据库副作用。若 handler 还调用支付、短信等外部系统，该调用仍需传稳定幂等键，或继续通过该服务自己的 Outbox 异步执行。

#### 4.2.5 一条完整时序

以“下单后预留库存”为例：

1. 订单服务在一个 DB 事务中插入 `orders(O1001)` 和 `outbox(E9001)`。
2. publisher 轮询或 CDC 读到 `E9001`，向 Kafka 发布；发送后崩溃导致 Kafka 中出现两份 `E9001`。
3. 库存服务第一次收到 `E9001`，在一个 DB 事务中插入 `inbox(inventory-service, E9001)` 并预留库存。
4. 第二次收到 `E9001` 时触发唯一键冲突，直接跳过；库存仍只预留一次。
5. 对账任务持续检查长时间未发布的 Outbox、异常 consumer lag 和业务状态差异，处理自动重试无法覆盖的配置错误、数据损坏或人工操作。

所以这里的 **effectively-once** 是：消息在传输层可能出现多次，但同一业务事件对每个消费者的最终业务效果只有一次。它不是“网络绝不重复”，也不是端到端无条件 exactly-once。

### 4.3 Saga 与补偿

Saga 把长事务拆成多个本地事务，每步成功后推进，失败时执行语义补偿。补偿不是数据库 rollback：退款可能失败、库存已被他人消费、链上交易无法删除，因此补偿本身也要幂等、可重试、可人工处理。

错误分类建议：

- **Transient**：超时、限流、临时不可用；指数退避 + jitter + 上限。
- **Conflict**：版本/nonce/余额变化；重新读取状态后决策，不盲重试旧请求。
- **Permanent**：参数、权限、业务拒绝；立即停止自动重试。
- **Unknown outcome**：请求超时但远端可能成功；先查询/对账，不能直接再执行。

### 4.4 DB + MQ 双写问题

具体解法要先看原子性边界：

1. **业务事实在关系数据库，消息发往 Kafka**：首选 [Transactional Outbox](#42-inbox--outbox)。业务更新和 Outbox 写入使用同一 DB 事务；轮询 publisher 或 CDC 至少一次投递；消费者用 Inbox / 幂等键；最后用监控和对账补足恢复闭环。这不是让 DB 与 Kafka 原子双写，而是彻底取消请求路径中的直接双写。
2. **输入、状态和输出都在 Kafka 内**：可使用 Kafka transaction 原子提交 consumed offsets 和输出 records，并让下游使用 `read_committed`。一旦同时更新外部数据库、支付接口或链上状态，就越出了 Kafka transaction 的保证范围。
3. **DB 和 MQ 都明确支持同一个 XA / 2PC 协调器**：理论上可做分布式原子提交，但会引入 coordinator、锁持有、阻塞恢复、可用性和运维成本；应按实际驱动与故障模型验证，不能把“框架有 `@Transactional`”当成已经覆盖 Kafka。
4. **MQ 提供事务消息 / 回查机制**：按该 MQ 的具体协议实现本地事务状态回查，本质上仍需要稳定业务 ID、可查询状态和幂等消费者，不能泛化成所有 MQ 的能力。

因此，常见的 DB + Kafka 落地答案是：**DB 本地事务写业务表和 Outbox → publisher / CDC 至少一次发 Kafka → Inbox 与消费业务同事务去重 → 对账补漏**。它将“跨两个系统的一次原子操作”改造成“两个本地原子操作 + 可重试消息 + 稳定身份”。

“Exactly-once”通常是业务效果，不是网络只传一次。网络可能丢、复制、乱序；系统通过身份、状态机和原子边界让重复不改变最终结果。

## 5. Temporal

### 5.1 Workflow 与 Activity

- **Workflow**：持久、可重放的协调逻辑；状态由 event history 恢复。
- **Activity**：与数据库、HTTP、链节点等外部世界交互的副作用，可配置 timeout/retry。
- **Signal**：异步发送事件给 workflow；不要求同步返回业务结果。
- **Update**：请求 workflow 验证并执行状态改变，可获得接受/完成结果。
- **Timer**：由服务端持久管理，worker 重启不会丢失。

Workflow 代码必须确定：相同 history 重放得到相同决策。不能直接读取当前时间、随机数、网络或非确定全局状态；应使用 SDK 提供的确定性 API 或 Activity。

### 5.2 重试、恢复与版本升级

Activity 可能“远端已成功、完成事件未记录”而被重试，因此它必须使用幂等 key。timeout 要区分 schedule-to-start、start-to-close、heartbeat 等阶段，避免把 worker 排队误判成外部调用慢。

升级 workflow 时，旧 execution history 会用新代码重放。使用 SDK 的 patch/versioning/worker deployment 机制兼容旧路径，不能直接删除历史分支。长寿命 workflow 还要设置 continue-as-new，控制 history 体积。

### 5.3 链上与跨链场景

链上确认 workflow 可建模：构造意图 → 分配 nonce → 签名广播 Activity → Timer/Signal 等确认 → reorg 回退 → 最终对账。Workflow 管协调，数据库/链仍是事实源；不要把几十万条区块日志塞进 history。

跨链消息、提现审批和延迟解锁适合 Temporal，因为它们有小时/天级等待、人工信号和补偿。但高吞吐 Scanner 不应每条 raw log 都启动复杂 workflow，先 canonicalize 和聚合再启动业务实例。

## 6. 可靠链上交易发送器

### 6.1 数据模型

建议核心表：

- `operation`：稳定 `operation_id`、业务类型、目标链、payload hash、状态。
- `nonce_lease`：`chain_id + sender + nonce` 唯一，记录租约、签名 hash 和替换序列。
- `tx_attempt`：每次签名/广播的 tx hash、费用、错误和发送节点。
- `receipt_observation`：block number/hash、status、confirm level、canonical 标记。
- `ledger_entry`：预期资产变化和对账结果。

业务去重键与 tx hash 分离。一项 operation 可能因费用替换产生多个 tx hash，但最终只能有一个 canonical attempt。

### 6.2 Nonce lease

单地址交易必须按 nonce 串行约束。发送器在数据库事务中分配 lease，不能依赖某个 RPC 的 `pending nonce` 作为唯一真相。多实例通过唯一键/CAS 防止重复分配；启动恢复时将本地 lease 与多 RPC 的 latest/pending 及链上 Receipt 对账。

nonce 前洞会阻塞后续交易。可替换卡住的旧交易，或提交同 nonce 自转账取消，但每次动作都需策略限额和审计，不能无限提高费用。

### 6.3 状态与错误分类

状态可为 `CREATED → SIGNED → BROADCAST → INCLUDED → SAFE → FINALIZED`，旁路包括：

- `DROPPED`：mempool 不再可见，但仍需防止晚入块。
- `REPLACED`：同 sender/nonce 的另一个 attempt 入块。
- `REORGED`：原 Receipt 所在 block 不再 canonical，回到 BROADCAST/RECONCILE。
- `STUCK`：长期未入块，允许有限 fee bump。
- `REVERTED`：canonical Receipt status 失败，通常是永久业务结果，不自动重放同 payload。

RPC timeout 属于 unknown outcome：应按 tx hash 和 sender/nonce 查询多个节点，再决定是否重发同一 raw transaction。重复广播相同 raw tx 通常安全，重新签名不同 nonce/数据则是新副作用。

### 6.4 安全与对账

- HSM/MPC 内签名，服务只传结构化 policy-approved 请求。
- 热钱包设余额/单笔/日限额，冷钱包补给；目标合约和 selector 白名单。
- 记录 unsigned payload、签名摘要、调用者、policy decision 和广播结果。
- 定时从链上 canonical 状态重算 nonce、余额和 operation，不能只信数据库状态。
- 对 ERC-20 还需处理 fee-on-transfer、rebasing、返回值异常和事件/余额差异。

## 7. 节点、RPC 与索引器

### 7.1 节点类型与同步

| 类型 | 能力 | 误区 |
| --- | --- | --- |
| Full | 验证链并服务近期状态 | 不一定能查询任意历史 state |
| Archive | 保存历史状态查询 | 不只是“保存全部 block”，成本主要在历史 state |
| Pruned | 删除可重建/不再服务的数据 | 各客户端 pruning 范围不同 |
| Snap Sync | 从可信证明化快照快速同步近期状态 | 完成快照不等于所有历史索引已就绪 |

Geth 常结合 trie/flat snapshot/freezer 等层次；Reth 采用 staged pipeline、数据库表和 static files 等设计。实现不同，但都必须核验同一共识结果。Ethereum/BSC 的协议状态树分别见 [01-ethereum.md](01-ethereum.md#4-状态与存储) 与 [02-bsc.md](02-bsc.md#6-存储与节点工程)。

### 7.2 高可用 RPC

- 按方法隔离：`eth_call`、trace、archive query、sendRawTransaction 的资源特征不同。
- 读请求可按 block hash 缓存；`latest` 缓存必须短且处理 reorg。
- 写广播到多个节点要去重并汇总错误，不因一个“already known”判断失败。
- quorum read 只减少单节点错误，多个节点若同客户端/同上游仍可能共同故障。
- 限流按 tenant + method + cost，不应把一次 trace 和一次 blockNumber 计为同成本。

### 7.3 Reorg Canonicalizer

索引表保存 `number, hash, parent_hash, canonical`。收到新 head：

1. 若 parent 等于当前 head hash，直接 append。
2. 否则沿新旧 parent 回溯找 common ancestor。
3. 旧分支标为 non-canonical，生成 UNDO 或按版本表回滚派生状态。
4. 新分支按高度重放，最终更新 head/finality 水位。

索引应可从 raw canonical facts 重建。把聚合余额直接覆盖且没有变更日志，会让 reorg 修复和离线重算非常困难。

### 7.4 多链充提与事件索引

多链平台把链差异封装为 adapter：finality、地址/交易编码、fee、nonce/UTXO、reorg 和 token 行为。统一层只使用标准化事件，不假设所有链都有 Ethereum 的 Receipt/nonce。

充提系统的完整链路是 Scanner → Canonicalizer → Deposit/Withdrawal state machine → Signer → Broadcaster → Confirmer → Reconciler。跨链桥的双边 ledger 见 [03-cross-chain-bridge.md](03-cross-chain-bridge.md#9-跨链后端系统设计)。

## 8. 背压、韧性与可观测性

### 8.1 背压和容量规划

容量至少估算 arrival rate、service rate、并发、队列等待、平均/峰值 payload 和恢复重放速度。系统在峰值后必须有 drain capacity，否则 lag 只会持续累积。

队列设置有界上限；满时按业务选择拒绝、降级、pause partition 或落盘，不能无限占内存。重试流量应有独立预算，避免故障时重试风暴压垮恢复中的依赖。

### 8.2 Timeout、Retry、Circuit Breaker

timeout 应来自端到端 deadline 预算，并向下游递减传播。只对 transient 且幂等的操作重试，使用指数退避、jitter 和最大次数。Circuit Breaker 在高失败率时快速拒绝，给依赖恢复窗口；Half-open 用少量探测恢复。

Hedged request 可能降低尾延迟，但会增加下游负载，只适合幂等读取并应延迟触发。Bulkhead 按链、租户或方法隔离连接池/worker，防止单热点耗尽全部资源。

### 8.3 SLI / SLO 与 Tracing

推荐 SLI：

- API availability、P50/P95/P99 latency、error ratio。
- Kafka consumer lag 和 oldest message age。
- node head lag、finality lag、reorg depth、RPC disagreement。
- transaction time-to-inclusion、stuck nonce、replacement rate。
- scanner canonical height、reconcile mismatch 和 manual queue age。

Tracing 使用 `trace_id / operation_id / message_id / tx_hash` 关联，但注意 tx hash 可替换。高基数属性避免直接成为 metric label；密钥、raw signature、用户隐私不能进入日志。

## 9. 高频面试问题与答题骨架

1. **如何保证 DB 更新后消息一定发出？** 请求事务只原子写业务表和带稳定 `event_id` 的 Outbox；publisher / CDC 至少一次发 Kafka；消费者把 Inbox 去重和业务更新放进同一事务；监控未发布记录并对账补漏。不要在请求路径直接双写 DB + MQ。
2. **Kafka exactly-once 能否覆盖支付接口？** 不能；Kafka transaction 边界内可原子处理 offsets/records，外部副作用需幂等键和对账。
3. **链上交易超时能否直接重试？** 先判 unknown outcome；查询 tx hash 和 nonce，优先重播相同 raw tx，避免新 nonce 双付。
4. **索引器如何处理 reorg？** hash/parent 链、common ancestor、UNDO/版本化、重放与 finality 水位。
5. **Temporal 为什么能恢复？** event history + deterministic replay；副作用在 Activity，Activity 自身仍需幂等。
6. **WAL 是否等于备份？** 否；WAL 支持崩溃恢复/PITR 的一部分，备份还需要独立快照、保留和恢复演练。
7. **节点追块慢如何定位？** 网络下载、验证、执行、state I/O、compaction、快照和硬件分段量化。

## 10. 容易混淆或说错的点

- Kafka 拼写是 **Kafka**，不是 Kafla。
- RWLock/RCU 解决内存并发，不解决跨服务一致性。
- “消息只发送一次”不是可现实依赖的网络保证；应追求业务效果幂等。
- Outbox 不等于 exactly-once；“Kafka 已发送、Outbox 未标记”的崩溃窗口会造成重复，必须有 Inbox 或业务幂等。
- Inbox 去重记录若不与业务更新处于同一 DB 事务，仍可能出现“已记为处理但业务未生效”或重复生效。
- WAL 不解决 DB + MQ/HTTP 的跨系统原子性。
- `fsync` 成功仍需考虑存储硬件/文件系统承诺和副本故障域。
- Temporal Workflow 的 durable 不等于 Activity 副作用 exactly-once。
- RPC 返回 tx hash 不代表交易入块，入块也不代表 final。
- Archive node 和 full node 的区别重点是历史 state 能力，不只是 block 文件多少。
- 固定等待 N 个块不是跨链通用 finality 策略。

## 11. 官方资料与延伸阅读

- [Linux RCU Documentation](https://www.kernel.org/doc/html/latest/RCU/)
- [bRPC DoublyBufferedData 源码](https://github.com/apache/brpc/blob/master/src/butil/containers/doubly_buffered_data.h)
- [PostgreSQL WAL](https://www.postgresql.org/docs/current/wal-intro.html)
- [Kafka Documentation](https://kafka.apache.org/documentation/) 与 [Kafka Design](https://kafka.apache.org/41/design/design/)
- [Debezium Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [Stripe Idempotent Requests](https://docs.stripe.com/api/idempotent_requests)
- [Temporal Workflows](https://docs.temporal.io/workflows) 与 [Activities](https://docs.temporal.io/activities)
- [Temporal Go Versioning](https://docs.temporal.io/develop/go/versioning)
- [Geth Databases](https://geth.ethereum.org/docs/fundamentals/databases)
- [Reth Storage](https://reth.rs/run/storage/)
- [Graph Node Ethereum Chain / Reorg 实现](https://github.com/graphprotocol/graph-node/blob/master/chain/ethereum/src/chain.rs)
- [OpenTelemetry Tracing](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
