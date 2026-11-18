# 最简自主智能体（Autonomous Agent）设计思路

## 一、设计目标

构建一个**不依赖大模型内置 function calling / tool_calls 能力**的自主智能体，让任意支持 Chat Completions 的模型（含不支持原生工具调用的模型）都能具备：

1. **自主决策**：自己判断下一步做什么，而不是被固定流程驱动
2. **工具调用**：读写文件、执行命令、SSH 远程操作、代码分析、批量分析等
3. **自我反思**：根据工具返回结果判断是否继续、是否需要换策略
4. **长任务不失控**：上下文超限时自动归档早期步骤，并可通过语义检索找回
5. **过程可观测**：每一步思考/行动/观察都实时推送到前端并持久化到数据库

核心取舍：**放弃原生 tool_calls，改用「markdown + step 代码块」的文本协议**。

---

## 二、整体架构

```
用户输入
   |
   v
[chat/index.ts  对话服务编排]
  - prepareChatContext：知识库检索 + 工程上下文 + 工具加载
  - 入口上下文压缩（ensureContextWithinLimit）
  - 创建助手消息占位符 + onPersist 持久化回调
  - 返回统一异步事件流（agent 事件 -> text 流 -> done）
   |
   v
[chat/agentLoop.ts  自主智能体主循环]
  while(true) {
    1. 读取信箱（异步通知注入）
    2. 请求前上下文压缩（三层防线）
    3. 调用 LLM（只返回 markdown content，不传 tools）
    4. stepParser 解析 step 代码块
    5. 逐个执行 step（think/executeTool/searchMemory/...）
    6. 结果合并回填为 user 消息 -> 进入下一轮
  }
   |
   v
[mcp/manager.ts  工具管理与执行]
  - 核心虚拟工具：think / searchMemory / searchStep / wait_for / finish_task / executeTool
  - executeTool 统一入口 -> 内置能力 + MCP Server 工具
  - 工具名校验（不存在时返回候选提示）
  - 工具向量索引（toolIndex.ts）-> 语义检索
```

## 三、核心设计一：Step 文本协议（替代原生 tool_calls）

### 3.1 为什么不用原生 tool_calls

- 部分模型（尤其国产模型、自部署模型）对 `tools` 参数支持不完整或行为不一致
- 原生 tool_calls 的并行调用、参数校验由 SDK 控制，难以插入自定义逻辑（如工具名校验、参数纠错）
- 文本协议让**模型输出完全可控**，任何 markdown 都能被拦截解析，便于做格式纠错与降级

### 3.2 协议格式

模型只输出 markdown content，工具调用写在 step 代码块中，每个块一个 JSON 对象：

```step
{"name":"executeTool","input":{"mcp_tool_name":"read_file","mcp_inputs":{"path":"D:\\a.ts"}}}
```

一次回复可包含多个 step 块，按顺序依次执行。


## 四、核心设计二：智能体主循环（agentLoop.ts）

### 4.1 循环结构

```
while (true) {
  round++
  1. 读取信箱（mailbox）-> 注入异步通知为 user 消息
  2. 请求前上下文压缩（ensureContextWithinLimit）
  3. 调用 LLM（temperature=0.3, max_tokens=32768, 禁用深度思考）
  4. 解析 step 块
     - 无 step 且无特殊 token -> 视为最终回答，break
     - 无 step 但有特殊 token -> 格式纠错重试
     - 有 step -> 逐个执行
  5. 执行 step（think / finish_task / wait_for / 核心工具）
  6. 结果合并为一条 user 消息回填 -> 下一轮
}
```

### 4.2 关键设计点

**1) 用户请求注入 system prompt**
每轮开始时把用户最新请求（去掉历史标签）追加到 system prompt 的「## 用户的当前请求」段落，确保多轮观察后智能体始终记得原始目标。

**2) step 协议说明动态注入**
把 `buildStepProtocolPrompt(toolsMarkdown)` 追加到 system prompt，包含：
- 工具调用格式硬性要求（禁止 DSML / tool_calls / XML 标签）
- 核心工具清单（think / searchMemory / searchStep / wait_for / finish_task / executeTool）
- 「先搜索再执行」的两步调用说明
- 「不要提前结束」的强制约束

**3) 信箱机制（mailbox）**
`contextInjector.ts` 提供会话信箱，支持异步任务完成后主动唤醒智能体：
- 每轮开始读取信箱，把待处理消息作为 `[系统通知 - source]` 注入
- `wait_for` 工具阻塞等待信箱消息（最长 300s），避免轮询导致上下文膨胀

**4) 核心工具 vs 普通工具**
- **核心工具**（可直接调用）：`executeTool` / `searchMemory` / `searchStep`
- **其他所有工具**（内置 + 外部 MCP）：**不能直接调用**，必须通过 `executeTool` 统一入口
- 若模型直接调用非核心工具，返回提示引导其走「先 searchMemory 查工具 -> 再 executeTool 执行」

**5) 反思阶段的信息获取判定**
若本轮只做了 `think` / `searchMemory` / `searchStep`（纯信息获取），回填消息会追加：
> 以上只是信息获取，任务尚未完成。请**立即输出 step 块**去真正执行下一步操作。

避免模型拿到信息就提前结束。

**6) finish_task 终止**
- 有 `summary` -> 直接作为最终回答（`final_text` 事件），退出循环
- 无 `summary` -> 让 LLM 继续输出最终回答

### 4.3 事件模型（types.ts）

```typescript
interface AgentEvent {
  type: 'think' | 'action' | 'observation' | 'error' | 'done' | 'final_text'
  round: number
  content?: string      // think / final_text 内容
  tool?: string         // 工具名
  args?: Record<string, unknown>
  result?: string       // 观察结果
  error?: string
  totalRounds?: number
}
```

`AgentEventStream` 是异步可迭代队列，支持实时推送（`emit`）与消费（`asyncIterator`），实现「智能体边执行、前端边展示」。

---

## 五、核心设计三：工具体系（mcp/manager.ts）

### 5.1 工具分层

```
- 核心虚拟工具（直接构建进工具列表，无需搜索）
    think          记录思考
    searchMemory   多维度搜索（项目/代码/修改记录/历史需求/工具/服务器）
    searchStep     语义检索当前对话已归档步骤
    wait_for       等待异步任务（信箱唤醒）
    finish_task    结束任务
    executeTool    统一执行入口

- 内置能力工具（由 manager 内部实现，经 executeTool 调用）
    batch_*        批量分析（大文件异步分块）
    ssh_*          SSH 远程执行
    file_profile   文件画像
    ...

- MCP Server 工具（内置 filesystem / shell + 外部 MCP）
    read_file / write_file / edit_file_lines / read_binary_file
    execute_command
    ...
```

### 5.2 统一执行入口 executeTool

```
executeTool(mcp_tool_name, mcp_inputs)
  - 1) dispatchBuiltinTool -> 内置能力（chain_/batch_/ssh_/file_profile）
  - 2) 遍历 MCP Server -> 匹配工具名并调用
  - 3) 都未匹配 -> 返回「工具不存在」+ 候选提示
```

兼容旧写法：`executeTool(action: "read_file", path: "xxx")` 参数平铺在顶层。

### 5.3 工具名校验（isKnownTool）

传入的工具名不存在时，**直接返回「工具不存在」而不是抛异常**，并给出：
- 模糊匹配候选（包含关系 + 公共前缀长度评分，取 Top5）
- 可用工具清单
- 引导调用 `searchMemory(scope: "tools")` 查看详情

这样 LLM 能自我纠正，而不是陷入错误循环。

### 5.4 工具语义索引（toolIndex.ts）

- 把「可执行工具」（MCP + 内置能力，不含核心虚拟工具）向量化后持久化到 `DATA_DIR/.tool-index.json`
- 引擎：内置 BGE-small 中文模型（纯本地推理）
- 索引条目含：名称、来源、描述、参数摘要、参数名、必填参数、关键词、向量
- 用「工具名+描述+参数」的 hash 作为指纹，判断是否需要重建
- `searchMemory(scope: "tools")` 走语义检索，按自然语言描述召回工具，无需精确匹配工具名

---

## 六、核心设计四：上下文长度管理（contextManager.ts）

### 6.1 设计原则

**在请求发出之前压缩**，而不是等 LLM 返回 400 错误后再补救。原实现只在返回后检查 `usage.total_tokens`，超限的那次请求已经失败了。

### 6.2 三层防线

```
第 1 层：单条消息硬截断
  - 超过 MAX_SINGLE_MESSAGE_CHARS（60000 字符）-> 保留头 70% + 尾 20%，中间标注已省略

第 2 层：请求前估算 + 循环归档早期步骤
  - 估算 token（ASCII 4字符约1token，中文 1字符约1token，保守高估）
  - 预算 = 阈值 - 工具定义 - 输出预留(32768)
  - 超预算 -> 按 assistant 消息分组归档，保留组数 6 -> 4 -> 2 -> 1
  - 归档内容已持久化到 execution_logs，可通过 searchStep 检索

第 3 层：归档后仍超限
  - 对剩余最大的 tool 消息激进截断（8000 -> 2000 -> 500 字符）
  - 截断 system prompt 中的知识库检索结果（保护「用户的当前请求」段落）
```

### 6.3 关键参数

| 参数 | 值 | 说明 |
|---|---|---|
| `MAX_TOTAL_TOKENS` | 262144 | 模型上下文总上限 |
| `CONTEXT_TRIGGER_RATIO` | 0.8 | 超过 80% 触发压缩 |
| `KEEP_RECENT_STEP_GROUPS` | 6 | 归档时保留最近步骤组数 |
| `MAX_SINGLE_MESSAGE_CHARS` | 60000 | 单条消息最大字符数 |
| `MIN_KEEP_GROUPS` | 1 | 归档下限（至少保留当前轮） |

### 6.4 归档提示注入

归档后向 system prompt 写入提示（替换上一次，避免堆叠）：

> 【步骤归档提示】由于上下文长度限制，本次对话中较早的 N 条步骤记录（第 1 ~ M 轮）已从上下文中移除，并完整保存到数据库。如需回忆请调用 searchStep 工具检索。


## 七、核心设计五：searchStep 步骤语义检索

### 7.1 问题背景

长任务中步骤过多，上下文放不下，早期步骤被归档出上下文。但智能体可能需要回忆「之前读到的文件内容」「执行过的命令结果」。

### 7.2 解决方案

1. **持久化**：每个 AgentEvent（think/action/observation/error）实时写入 `execution_logs` 表，含 sessionId、messageId、round、toolName、toolArgs、result、error
2. **归档**：上下文超限时把早期步骤移出上下文（数据库里仍完整保留）
3. **检索**：提供 `searchStep` 工具，按语义检索**当前对话**已归档的步骤

### 7.3 searchStep 实现（search.ts -> searchStepDB）

```typescript
searchStep(query, sessionId, limit=8, roundTo?)
  -> dao.searchExecutionLogsBySession(query, sessionId, limit, roundTo)
  -> 格式化输出：轮次 + 工具名 + 参数(截断500) + 内容(截断1500) + 结果(截断2000)
```

- `sessionId` 由 `callTool` 自动注入（不污染调用方参数）
- `roundTo` 可限定只检索更早轮次
- 检索算法：关键词分词 + 评分（关键词长度加权 + 出现次数 + 二元组子串加分）

### 7.4 与 searchMemory 的区别

| 工具 | 范围 | 用途 |
|---|---|---|
| `searchMemory` | 项目/代码/修改记录/历史需求/工具/服务器 | 跨会话的长期记忆 |
| `searchStep` | **当前对话**已归档步骤 | 单次长任务的短期记忆找回 |

---

## 八、核心设计六：流式编排与持久化（chat/index.ts）

### 8.1 三阶段事件流

```
阶段一 agent：实时转发智能体事件（think/action/observation）
              - final_text 事件 -> 累积为最终回答
阶段二 text ：智能体未 finish_task -> 让 LLM 生成最终 markdown 回答（流式）
阶段三 done ：结束
```

### 8.2 后端直接持久化

- 创建助手消息占位符（`dao.createMessage`）
- `onPersist` 回调把每个 AgentEvent 实时写入 `execution_logs`
- 流式文本增量更新助手消息内容（每 500 字符或遇换行保存一次）
- 最终内容做 `stripDsmlResidue` 兜底清理

## 九、系统提示词设计

系统提示词模板位于 `agent-system.md`，动态加载（基于 mtime 缓存，文件修改后自动热更新）。

### 9.1 组成结构

```
agent-system.md（基础模板）
  + 工程上下文（getProjectContext）
  + 可用服务器列表（dao.listServers）
  + 知识库检索上下文（retrievalContext / chainContext）
  + step 协议说明（buildStepProtocolPrompt）
  + 工具清单 markdown（buildToolsMarkdown）
  + 用户的当前请求（agentLoop 每轮注入）
  + 步骤归档提示（contextManager 归档后注入）
```

### 9.2 提示词要点

- **工作模式**：决策 -> 行动 -> 反思 -> 等待 -> 结束
- **格式硬性要求**：只输出 markdown，工具调用只能写 step 代码块，严禁 DSML / tool_calls / XML 标签
- **工具调用方式**：核心工具可直接调用，其他工具必须「先 searchMemory 搜索 -> 再 executeTool 执行」
- **不要提前结束**：拿到工具信息后必须紧接着执行，不能只做说明

> 完整系统提示词见同目录 `系统提示词-agent-system.md`。

---

## 十、设计要点总结

| 设计 | 解决的问题 |
|---|---|
| Step 文本协议 | 摆脱原生 tool_calls 依赖，兼容任意模型，输出完全可控 |
| 多格式解析 + DSML 兼容 | 模型误用格式时仍能正确执行，不把特殊 token 展示给用户 |
| 格式纠错重试 | 模型回退到原生格式时自动纠正 |
| 核心工具 + executeTool 统一入口 | 工具数量多时不撑爆提示词，按需语义检索 |
| 工具名校验 + 候选提示 | 工具名拼错时 LLM 能自我纠正 |
| 请求前上下文压缩（三层防线） | 避免超限请求 400 失败 |
| 步骤归档 + searchStep | 长任务上下文可控，早期步骤可找回 |
| 信箱 + wait_for | 异步任务不轮询，避免上下文膨胀 |
| 事件流 + 实时持久化 | 过程可观测、可回放、可断点续看 |
| 用户请求注入 system prompt | 多轮观察后不丢失原始目标 |

---
