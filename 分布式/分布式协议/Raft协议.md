## Raft 协议

### **1. 设计目标与背景**

- Raft 是为了解决 Paxos“难懂、难实现”的问题而提出的一种**可理解的一致性算法**。
- 主要目标：
  - 与 Paxos 等价的安全性（强一致性、线性一致）。
  - 更易解释、更易实现、更易教学。
- 广泛应用：
  - `etcd`、`Consul`、`TiKV` 等系统都采用 Raft 存储集群配置与用户数据。

### **2. 系统模型与前提**

- **集群**由多个节点（一般为 3/5/7 个）组成。
- 使用日志复制状态机模型：
  - 所有节点维护一份操作日志（log）。
  - 每个节点将日志按顺序应用到状态机上，保证状态机行为一致。
- 故障模型：
  - 节点可能随时宕机、重启，但不恶意作恶（非拜占庭）。
  - 网络可能丢包、乱序、重复、延迟。

### **3. 角色、任期与状态**

- **角色**（三种）：
  - **Leader（领导者）**：处理所有客户端请求、负责日志复制。
  - **Follower（跟随者）**：被动响应 Leader 和 Candidate 的请求，不主动发起 RPC。
  - **Candidate（候选者）**：在选举中发起投票，请求成为 Leader。
- **Term（任期）**：
  - 逻辑上的单调递增的“时代编号”。
  - 每次选举产生新的 Leader，即进入新的任期。
- **重要状态**：
  - `currentTerm`：本节点已知的最新任期号。
  - `votedFor`：本任期中本节点投给哪个候选者。
  - `log[]`：日志条目，每条包含 `(term, index, command)`。

### **4. Leader 选举**

#### 4.1 基本过程

1. 所有节点初始为 Follower。
2. 若 Follower 在 `election timeout` 时间内没有收到 Leader 的心跳（`AppendEntries`）：
   - 转为 Candidate，`currentTerm++`，给自己投票。
   - 向其他节点发送 `RequestVote` RPC。
3. 其他节点接到投票请求时：
   - 若请求中的任期号 `<` 自己的 `currentTerm`，拒绝。
   - 若自己本任期尚未投票，且候选人的日志**不落后于**自己，则投票给它。
4. 候选者获得**多数派**选票后：
   - 成为本任期 Leader。
   - 立即向所有 Follower 发送心跳（空的 `AppendEntries`）以确立领导地位。

#### 4.2 日志“新旧”判断（防止旧 Leader 上位）

- Raft 定义“日志不落后于”：
  1. 比较最后一条日志的 `lastLogTerm`，大的更新。
  2. 若 `lastLogTerm` 相同，比较 `lastLogIndex`，大的更新。
- 这样可以确保选出的 Leader 的日志至少**不比任何投票给它的节点落后**。

### **5. 日志复制**

#### 5.1 写入流程（强 Leader）

1. 客户端将命令发送给 Leader。
2. Leader 将命令追加到自己的日志末尾（未提交）。
3. Leader 并行向所有 Follower 发送 `AppendEntries(term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit)`。
4. Follower 检查：
   - 如果本地在 `prevLogIndex` 处的日志条目的 `term` 不等于 `prevLogTerm`：
     - 表示日志不一致，拒绝本次追加。
   - 否则：
     - 删除本地从 `prevLogIndex+1` 开始与新 entries 冲突的条目。
     - 追加新的 entries。
5. 如果 Leader 收到**多数派** Follower 对该日志条目的成功响应：
   - Leader 将日志条目标记为**已提交（committed）**。
   - 更新 `commitIndex`，并在本地状态机中执行该命令。
   - 通过后续心跳/`AppendEntries` RPC 告知 Follower 最新的 `leaderCommit`。
6. Follower 在得知某条日志已被提交后，也会应用这条日志到本地状态机。

#### 5.2 日志一致性与冲突解决

- Raft 要求：
  - 任意两个已提交的日志前缀必须完全相同。
- 冲突解决策略：
  - 若 Follower 在 `prevLogIndex` 处的 term 不匹配：
    - 表明从这个位置往后存在分歧。
    - Follower 要删除从冲突位置开始的所有本地日志。
  - Leader 会向前回退 `prevLogIndex` 重试，直到双方的前缀一致。
- 这样可以确保最终所有节点的日志与 Leader 保持**相同前缀**。

### **6. 安全性（Safety）保证**

- **Leader Completeness Property（领导者完备性）**：
  - 一旦某条日志在某个任期被提交，则该日志必定出现在之后所有任期的 Leader 日志中。
- **日志匹配特性（Log Matching Property）**：
  - 若两条日志在同一 `index` 具有相同的 `term`，则它们之前的所有日志也完全相同。
- **状态机安全性（State Machine Safety）**：
  - 如果某节点状态机将某条日志条目应用到某个索引位置，那么其他任何节点不会在该索引位置应用不同的日志条目。

### **7. 成员变更（集群扩缩容）**

- Raft 使用 **联合一致（Joint Consensus）** 方案来变更集群成员：
  - 在成员变更过程中，有一个过渡配置，包含旧配置 + 新配置的所有节点。
  - 任何日志提交都必须在旧配置和新配置都达到多数派。
  - 变更流程大致为：
    1. 提交一个包含新配置的日志（联合配置）。
    2. 等联合配置稳定后，再提交一个只包含新配置的日志。
- 这样可以保证在配置变更期间不会产生“两个 Leader”或丢失提交日志。

### **8. Raft vs Paxos（简要对比）**

- **模型与表达方式**：
  - Paxos：以“投票”“提案”为抽象，偏数学与理论。
  - Raft：以“主从复制 + 日志 + 任期 + 选举”为抽象，贴近工程实践。
- **Leader 角色**：
  - Raft 为**强 Leader**模型，所有客户端写请求必须通过 Leader。
  - Paxos（尤其 Multi-Paxos）通常也会选出 Leader，但表达上不如 Raft直观。
- **实现难度**：
  - Raft 的模块划分清晰（选举、日志复制、成员变更），每部分有明确定义。
  - 绝大多数工程团队倾向选择 Raft 作为一致性协议基础（尤其是新的系统）。

### **9. 工程实践要点**

- **选举超时（Election Timeout）配置**：
  - 通常为数百毫秒到几秒，且在不同节点上使用**随机化**时间，避免选举冲突。
- **心跳间隔（Heartbeat Interval）**：
  - 通常远小于选举超时（比如 50~150ms 与 500~1500ms），保证 Leader 能稳定宣示。
- **持久化**：
  - `currentTerm`、`votedFor` 以及日志必须在响应客户端前持久化（写盘/刷盘）。
- **快照（Snapshot）**：
  - 日志增长过大时进行快照，将状态机当前状态持久化，并丢弃之前的日志片段。
  - 节点恢复或新加入时，可通过快照 + 后续日志快速追上进度。
- **常见实现**：
  - Go 语言生态：`etcd/raft` 实现广泛使用。
  - Java 生态：有基于 Netty + Raft 的实现，用于内部配置管理、分布式协调等。

### **10. 面试/笔试要点总结**

- **必须掌握**：
  - 三种角色（Leader/Follower/Candidate）。
  - 任期（Term）和选举流程（RequestVote）。
  - 日志复制流程（AppendEntries）以及如何处理冲突。
- **可扩展点**：
  - Leader Completeness、Log Matching 等安全性性质。
  - 成员变更（Joint Consensus）的基本思路。
  - 与 Paxos 的区别：可理解性、工程友好性、更适合日志复制场景。

