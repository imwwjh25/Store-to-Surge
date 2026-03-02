### 一、先明确：ReAct 模式下死循环的典型成因

ReAct 模式的核心是 “Reason（推理）→ Action（行动）→ Observation（观察）→ 再 Reason” 的闭环，死循环本质是这个闭环卡在 “无效推理 - 重复行动 - 无新观察” 的循环里，常见原因有：

1. **目标模糊 / 不可达**：Agent 无法拆解目标（如 “整理文档” 无具体规则），或目标本身无法完成（如 “查询不存在的数据”），反复执行相同推理；
2. **反馈缺失 / 无新信息**：Action 的 Observation 无新内容（如每次调用工具都返回 “无结果”），Agent 无法更新状态，只能重复之前的思考；
3. **推理逻辑缺陷**：Agent 的 Reason 环节无法识别 “已尝试过该 Action”，或缺乏 “失败重试次数限制”，无限重复相同 Action；
4. **状态管理失效**：未记录历史思考 / 行动轨迹，Agent 无法感知 “自己在循环”（如反复调用同一个工具、问同一个问题）。

### 二、第一步：如何识别死循环？（可量化的判断规则）

要解决死循环，首先要能**精准识别循环状态**，避免误判（比如 Agent 只是正常多步推理），常用可落地的识别规则：








|   识别维度   |                         具体判断条件                         |
| :----------: | :----------------------------------------------------------: |
| 重复行动检测 | 连续 N 次（如 3 次）执行相同的 Action（如重复调用同一个工具、传入相同参数）； |
| 推理文本重复 | Reason 环节生成的思考文本相似度≥90%（如连续 3 轮推理几乎一样）； |
| 无新观察反馈 | 连续 M 次（如 2 次）Observation 返回 “无结果”“失败”“相同内容”，无新信息； |
|   步数超限   | 整体思考步数超过阈值（如 10 步）仍未接近目标（可结合目标完成度判断）； |
|  状态无更新  | Agent 的核心状态（如已尝试的工具、已获取的信息）连续多轮无变化； |

**工程化实现示例**：

为 Agent 维护一个「历史轨迹缓存」，记录每一轮的 Reason 文本、Action 类型 + 参数、Observation 结果，通过以下逻辑判断循环：









```
# 伪代码：识别重复Action循环
def is_dead_loop(history, repeat_threshold=3, step_threshold=10):
    # 1. 步数超限直接判定
    if len(history) >= step_threshold:
        return True
    # 2. 检测连续重复的Action
    if len(history) < repeat_threshold:
        return False
    # 取最后repeat_threshold轮的Action
    last_actions = [h["action"] for h in history[-repeat_threshold:]]
    # 检查是否所有Action完全相同
    return all(act == last_actions[0] for act in last_actions)
```

### 三、核心解决策略：从 “识别” 到 “打断 + 引导”

识别到死循环后，不能只简单 “终止”，而是要**主动干预**，引导 Agent 跳出循环，核心策略按优先级排序：

#### 1. 基础策略：硬限制（兜底）

最直接的工程化手段，避免 Agent 无限消耗资源，适合所有场景：

- **步数限制**：预设最大思考步数（如 10 步），达到阈值直接终止循环，返回 “无法完成目标” 或兜底结果；
- **重试次数限制**：对单个 Action 设置最大重试次数（如同一工具最多调用 3 次），超次数则禁止再次调用该 Action；
- **超时限制**：对整个 ReAct 流程设置超时（如 30 秒），超时后终止并反馈。

#### 2. 核心策略：轨迹感知 + 去重（避免重复行动）

让 Agent “记住自己做过什么”，从根源上避免重复推理 / 行动：

- 记录历史行动集 ：为 Agent 维护一个```visited_actions```集合，存储已执行的 Action（如```(工具名, 参数)```），Reason 环节先检查：



✅ 未执行过 → 正常推理；



❌ 已执行过 → 跳过该 Action，生成新的推理路径；

- 历史轨迹注入 Prompt ：每次 Reason 前，将历史思考 / 行动 / 观察结果注入 Prompt，明确提示 Agent：

  > 你已尝试过以下行动且无有效结果：[调用工具 A (参数 1)→返回无结果，调用工具 A (参数 2)→返回无结果]，请避免重复尝试，尝试其他工具或调整参数。




#### 3. 进阶策略：主动干预 + 重规划（引导新路径）

识别到循环后，不终止而是**强制 Agent 重新拆解目标**，生成新的推理路径：

- **目标重拆解**：当检测到循环，触发 “目标分解” 逻辑，将原目标拆分为更小的子目标（如原目标 “查询并分析数据”→拆分为 “1. 确认数据是否存在；2. 若存在则查询；3. 分析数据”）；
- **工具切换引导**：若 Agent 重复调用某工具失败，Prompt 提示尝试替代工具（如 “调用数据库查询失败，可尝试调用缓存查询工具”）；
- **参数调整建议**：若因参数错误导致循环，Agent 分析 Observation 反馈，自动调整参数（如 “查询时间范围过大返回无结果，建议缩小时间范围为近 7 天”）；
- **人工介入（可选）**：对关键场景，循环达到阈值后暂停，推送人工确认（如 “Agent 无法完成目标，是否需要人工指导？”）。

#### 4. 兜底策略：回退 + 重启（重置思考状态）

若上述策略无效，可让 Agent “回退” 到之前的有效状态，重新开始推理：

- **状态回退**：保存每一轮的思考状态（如第 3 步的 Reason 结果），循环时回退到最近的 “有效状态”（如 Observation 有新信息的步骤），重新推理；
- **轻量重启**：清空当前历史轨迹，保留核心目标，让 Agent 重新开始 ReAct 流程（避免完全重启丢失上下文）。

### 四、落地实现示例（Python 伪代码）

以 “ReAct Agent 调用工具查询数据” 为例，完整实现 “识别循环 + 打断 + 重规划”：







```
class ReActAgent:
    def __init__(self, max_steps=10, repeat_threshold=3):
        self.max_steps = max_steps  # 最大步数限制
        self.repeat_threshold = repeat_threshold  # 重复Action阈值
        self.history = []  # 存储每轮轨迹：[{"reason":..., "action":..., "observation":...}]
        self.visited_actions = set()  # 已执行的Action集合

    def react_loop(self, goal):
        step = 0
        while step < self.max_steps:
            # 1. Reason：推理下一步行动（注入历史轨迹避免循环）
            reason = self._reason(goal, self.history)
            # 2. Action：生成行动（检测重复Action）
            action = self._generate_action(reason)
            action_key = (action["tool"], tuple(action["params"].items()))  # 标准化Action
            
            # 检测死循环：重复Action或推理文本重复
            if self._is_dead_loop() or action_key in self.visited_actions:
                # 干预策略：引导重规划
                reason = self._reprompt_for_replan(reason, self.history)
                action = self._generate_action(reason)  # 生成新Action
                action_key = (action["tool"], tuple(action["params"].items()))
            
            # 3. 执行Action并记录
            self.visited_actions.add(action_key)
            observation = self._execute_action(action)
            # 4. 记录轨迹
            self.history.append({
                "reason": reason,
                "action": action,
                "observation": observation
            })
            
            # 5. 检查目标是否完成
            if self._is_goal_achieved(observation, goal):
                return self._format_result(observation)
            
            step += 1
        
        # 步数超限，返回兜底结果
        return f"无法完成目标{goal}，已达到最大思考步数{self.max_steps}"

    def _is_dead_loop(self):
        # 实现前文的死循环识别逻辑
        if len(self.history) < self.repeat_threshold:
            return False
        last_actions = [h["action"] for h in self.history[-self.repeat_threshold:]]
        return all(act == last_actions[0] for act in last_actions)

    def _reprompt_for_replan(self, old_reason, history):
        # 生成重规划的Prompt，引导Agent跳出循环
        repeat_actions = [h["action"] for h in history[-self.repeat_threshold:]]
        reprompt = f"""
        你当前陷入思考循环，已连续{self.repeat_threshold}次执行相同行动：{repeat_actions[0]}，且未获得新的有效信息。
        请放弃该行动路径，重新分析目标{goal}，尝试以下方向：
        1. 检查是否目标本身无法完成；
        2. 尝试调用其他工具（当前已调用：{list(self.visited_actions)}）；
        3. 调整行动参数，避免重复；
        原推理：{old_reason}
        新推理：
        """
        return self._llm_generate(reprompt)  # 调用LLM生成新的推理
```

### 五、额外优化：从根源减少死循环概率

除了被动干预，还可以在 Agent 设计阶段提前规避：

1. **目标结构化**：要求用户输入 “可拆解、可验证” 的目标（如 “查询 2024 年 1 月订单数据并统计金额”，而非 “整理订单数据”）；
2. **工具元信息配置**：为每个工具配置 “失败重试策略”“替代工具列表”，Agent 可直接读取；
3. **状态可视化**：记录 Agent 的思考轨迹，便于定位循环原因（如某工具返回结果格式错误，导致 Agent 无法解析）；
4. **冷启动优化**：用历史成功案例训练 Agent，让其学习 “如何识别循环、如何切换路径”。

------

### 总结

1. **死循环识别**：核心是监控「重复 Action、无新观察、步数超限」，通过历史轨迹缓存实现可量化判断；
2. **核心解决策略**：先靠 “步数 / 重试次数硬限制” 兜底，再通过 “轨迹去重 + 历史注入” 避免重复，最后用 “重规划 / 回退” 引导新路径；
3. **工程化关键**：为 Agent 维护「历史轨迹 + 已执行 Action 集」，在 Reason 环节主动干预，而非等到循环发生后终止。