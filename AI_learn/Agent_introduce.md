|阶段|核心问题|你最终要理解什么|
|--|--|--|
|**第一阶段：理解 Agent**|Agent 到底是什么？|Agent、LLM、Workflow、Tool、Memory、Environment 的关系|
|**第二阶段：Harness**|Agent 靠什么运行？|Agent Loop、Context、State、Tool Execution、Control、Runtime|
|**第三阶段：Tools**|Agent 如何与世界交互？|Tool Design、Function Calling、API、Browser、Code、MCP、Tool Selection|
|**第四阶段：Memory**|Agent 如何“记住”？|Working Memory、Episodic Memory、Semantic Memory、Long-term Memory、RAG|
|**第五阶段：Planning & Reasoning**|Agent 如何完成复杂任务？|Planning、Decomposition、Reflection、Replanning、Reasoning、Multi-step Execution|
|**第六阶段：Multi-Agent**|多个 Agent 如何协作？|Roles、Delegation、Communication、Coordination、Supervisor、Agent Teams|
|**第七阶段：Production Agent**|怎么把 Agent 真正做成可靠的系统？|Evaluation、Observability、Security、Guardrails、Reliability、Cost、Latency、Deployment|

<br/>

# 一、理解 Agent

## Agent 的基本认知模型

```
           User
            │
            ↓
          Task
            │
            ↓
      ┌───────────┐
      │   Model   │
      └─────┬─────┘
            │
      决定下一步
            │
            ↓
       Tool Call
            │
            ↓
      ┌───────────┐
      │  Runtime  │
      └─────┬─────┘
            │
         执行工具
            │
            ↓
       Tool Result
            │
            ↓
      更新 State
            │
            ↓
      更新 Context
            │
            ↓
          Model
            │
            ↓
          ……
```

## 最重要的几个概念

① LLM 不是 Agent

```
LLM = 负责生成/决策
Agent = LLM + Runtime + Tools + Loop
```

② Tool Calling 让模型能够“行动”

```
模型：
我要运行 npm test

Runtime：
好的，我真的运行 npm test
```

③ Agent Loop 让模型可以持续行动

```
思考
→ 行动
→ 观察
→ 思考
→ 行动
→ 观察
→ …
```

④ State 是 Agent 的工作状态

```
做过什么
现在是什么状态
遇到了什么问题
下一步是什么
```

⑤ Context 是模型当前能看到的信息

```
System
+
Instructions
+
History
+
Files
+
Tool Results
+
State
```

## 第一阶段的核心 Mental Model

<br/>

```
          ┌─────────────────────┐
          │        Agent        │
          │                     │
          │  ┌───────────────┐  │
          │  │     Model     │  │
          │  │               │  │
          │  │  决定下一步     │  │
          │  └───────┬───────┘  │
          │          │          │
          │      Tool Call      │
          │          ↓          │
          │  ┌───────────────┐  │
          │  │     Tools     │  │
          │  │               │  │
          │  │ 文件/终端/Git  │  │
          │  └───────┬───────┘  │
          │          │          │
          │      Tool Result     │
          │          ↓          │
          │  ┌───────────────┐  │
          │  │     State     │  │
          │  │               │  │
          │  │ Agent 当前状态 │  │
          │  └───────┬───────┘  │
          │          │          │
          │       Context       │
          │          │          │
          │          ↓          │
          │        Model        │
          │          │          │
          │          └──→ ……    │
          └─────────────────────┘
```

# 第二阶段：Harness

```
Harness 是围绕 Agent 的运行时控制层，
它负责把模型、上下文、工具、状态、环境以及执行控制组织成一个持续运行的 Agent Loop。
```

换句话说：

> Harness 本身不是 Agent 的“大脑”。

它更像是：

> Agent 的运行框架 / Runtime / Operating Environment。

<br/>

## Harness 的第一层：Loop Control

Agent 的核心其实是一个 Loop

```
        ┌─────────────┐
        │    User     │
        └──────┬──────┘
               ↓
        ┌─────────────┐
        │    Context  │
        └──────┬──────┘
               ↓
        ┌─────────────┐
  ┌────→│     LLM     │
  │     └──────┬──────┘
  │            ↓
  │      Need Tool?
  │        /       \
  │      Yes        No
  │       ↓          ↓
  │    Tool       Final Answer
  │       ↓
  │    Result
  │       ↓
  └──── Context
```

## 第二层：Context Management

Context Assembly：

```
System Instructions
+
User Request
+
Relevant History
+
Current State
+
Available Tools
+
Latest Tool Result
```

## 第三层：State

Context ≠ State。

State 是：

```
Agent 当前处于什么状态。
```

因为复杂 Agent 往往不是：

```
想 → 做 → 结束
```

而是：

```
Task
 ↓
Plan
 ↓
Research Revenue
 ↓
Research Data Center
 ↓
Research Guidance
 ↓
Compare
 ↓
Write
 ↓
Done
```

Harness 需要知道：

```
现在进行到哪一步了？
```

## 第四层：Tool Execution

模型说：

> {
  "tool": "search",
  "query": "NVIDIA data center revenue"
}

模型不会真的执行搜索。

Harness 收到这个请求之后：

```
LLM
 ↓
Tool Call
 ↓
Harness
 ↓
Tool Registry
 ↓
Search Tool
 ↓
Internet
 ↓
Result
 ↓
Harness
 ↓
LLM
```

所以 Tool 实际上是：

> 由 Harness 执行，而不是由 LLM 执行。

Harness 的真正位置：

```
                AGENT SYSTEM
 ─────────────────────────────────────────
             ┌───────────┐
             │    LLM    │
             │           │
             │ Reasoning │
             └─────┬─────┘
                   │
                   ▼
          ┌─────────────────┐
          │     HARNESS     │
          │                 │
          │  Agent Loop     │
          │  Context        │
          │  State          │
          │  Tool Runtime   │
          │  Error Handling │
          │  Retry          │
          │  Termination    │
          └────────┬────────┘
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Browser       API        Code
```

Harness 的内部架构

```
                     Harness
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
  Context Manager   Agent Loop      State Manager
       │               │                │
       ▼               ▼                ▼
   Prompt Builder   Controller      Checkpoint
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Tool Runner   Retry       Termination
          │
          ▼
     Tool Registry
```

## Mini Harness

```python
class AgentHarness:

    def __init__(
        self,
        llm,
        tools,
        context_manager,
        state,
        policy,
        checkpoint
    ):
        self.llm = llm
        self.tools = tools
        self.context = context_manager
        self.state = state
        self.policy = policy
        self.checkpoint = checkpoint

    def run(self, task):

        self.state.start(task)

        while not self.policy.should_stop(self.state):

            context = self.context.build(
                self.state
            )

            response = self.llm(context)

            if response.is_final:

                self.state.complete()

                self.checkpoint.save(
                    self.state,
                    self.context
                )

                return response.content

            if response.is_tool_call:

                result = self.execute_tool(
                    response
                )

                self.context.add(
                    response
                )

                self.context.add(
                    result
                )

                self.state.step += 1

                self.checkpoint.save(
                    self.state,
                    self.context
                )

        return self.handle_termination()
```

<br/>

## Context Engineering

### Context Assembly

因此每一次调用 LLM 前，Harness 都应该做：

重新组装这一次模型真正需要看到的上下文。

```
                 Agent State
                      │
                      ▼
              Context Builder
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
   System Prompt   Recent History   Relevant Info
        │             │             │
        └─────────────┼─────────────┘
                      ↓
                Current Context
                      ↓
                     LLM
```

- 第一种方式：Full History
- 第二种：Sliding Window：只保留最近 N 个消息。
- 第三种：Summary + Raw Facts
  ```
  Summary:
  Data Center revenue increased strongly.
  
  Important Facts:
  Q1 = $10B
  Q2 = $12B
  Growth = 20%
  ```

- 第四种：Relevant Context

## Harness 完整分叉图

```
                                    Harness
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
  Context Manager               Agent Loop                     State Manager
        │                             │                             │
  ┌─────┼──────┐               ┌──────┼──────┐               ┌──────┼──────┐
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Collect Prune Compose      Controller Tool Runner          Schema  Store  Snapshot
  │     │      │               │      │      │               │      │      │
  ▼     ▼      ▼               │      ▼      │               ▼      ▼      ▼
Inputs Truncate SystemPrompt   │ Tool Registry  │          Shape  Memory Checkpoint
History Summarize ToolDefs     │      │         │          Types  Persist Restore
ToolResults Tokenize Messages  │      ▼         │          Defaults Cache  Rollback
                                │ Tool Definition│
                                │      │         │
                                ▼      ▼         ▼
                           Guardrails Retry  Termination
                                │      │         │
                                └──────┼─────────┘
                                       ▼
                                  Error Handler
                                       │
                                       ▼
                                    Runtime
                              （底座，支撑整个 Harness）
```

### ① Context Manager —— 管"LLM 这一轮看到什么"

|层级|子概念|一句话介绍|
|--|--|--|
|收集|**Collect**|从各处抓取上下文：用户输入、历史、工具结果|
||├─ Inputs|用户本轮输入 + 初始指令|
||├─ History|历史对话消息（多轮上下文）|
||└─ ToolResults|之前工具调用的输出|
|裁剪|**Prune**|超窗口时决定丢什么、留什么|
||├─ Truncate|直接截断（丢最旧/超长）|
||├─ Summarize|压缩成摘要（比截断更保信息）|
||└─ Tokenize|按 token 精确计量防超窗|
|拼装|**Compose**|组装成最终 prompt|
||├─ SystemPrompt|角色/规则/约束（最高优先级）|
||├─ ToolDefs|工具 name/desc/schema|
||└─ Messages|消息序列（system+历史+输入）|

### ② Agent Loop —— 管"Agent 如何循环跑"

|层级|子概念|一句话介绍|
|--|--|--|
|驱动|**Controller**|决策大脑：下一步走分支/工具/输出（Control 核心）|
|工具|**Tool Runner**|把 LLM 的 tool_call 变成真实函数/命令/API 调用|
||**Tool Registry**|工具登记表：名字、schema、描述|
||**Tool Definition**|每个工具的参数 JSON Schema、返回值契约|
|约束|**Guardrails**|安全权限：敏感操作要不要确认、谁能做|
|容错|**Retry**|失败重试：次数、退避、是否降级|
|停止|**Termination**|终止判断：完成/步数上限/用户中断|
|兜底|**Error Handler**|兜底错误处理：可重试→重试，不可→降级/上报|

### ③ State Manager —— 管"Agent 记住什么"

|层级|子概念|一句话介绍|
|--|--|--|
|结构|**Schema**|定义 State 长什么样|
||├─ Shape|数据结构（messages/variables/step）|
||├─ Types|字段类型|
||└─ Defaults|初始默认值|
|存储|**Store**|State 存在哪|
||├─ Memory|内存态（快，进程退即丢）|
||├─ Persist|落盘持久化（重启不丢）|
||└─ Cache|缓存层（加速高频读）|
|恢复|**Snapshot**|状态可恢复性|
||├─ Checkpoint|关键节点拍快照|
||├─ Restore|断点恢复（如 `--continue`）|
||└─ Rollback|回滚到之前状态重来|

#  第三阶段：Tools —— Agent 如何与世界交互

```
                                    Tools
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
  Tool Design                   Function Calling              Tool Selection
        │                             │                             │
  ┌─────┼──────┐               ┌──────┼──────┐               ┌──────┼──────┐
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Define Describe Schema       Emit    Parse  Execute        Match   Rank   Route
  │     │      │               │      │      │               │      │      │
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Name  Purpose Params          tool_call args   call         keyword score  intent
Usage  Doc     Return         request JSON    dispatch      semantic filter fallback
  │
  ▼
  ┌───────────────┴────────────────┐
  ▼                                ▼
Tool Types                    Tool Ecosystem
  │                                │
  ├─ API ────────────────┐         ├─ Registry ──── 登记/发现
  ├─ Browser ────────────┤         ├─ MCP ───────── 跨应用统一协议
  ├─ Code ───────────────┤         └─ SDK/Plugin ── 封装/扩展
  └─ MCP ────────────────┘
```

### ① Tool Design —— 工具怎么设计（让 LLM 会用）

|层级|子概念|一句话介绍|
|--|--|--|
|定义|**Define**|明确工具是干什么的|
||├─ Name|工具名（LLM 用它来引用，要语义清晰）|
||├─ Usage|一句话用途说明|
||└─ Doc|详细文档/示例|
|描述|**Describe**|让 LLM 理解何时用、怎么用|
||├─ Purpose|触发场景描述（什么情况该调它）|
||└─ Doc|使用说明（含示例）|
|契约|**Schema**|工具调用的"接口契约"|
||├─ Params|参数定义（JSON Schema：类型、必填、约束）|
||└─ Return|返回值契约（结构、错误格式）|

### ② Function Calling —— LLM 怎么发出并执行工具调用

|层级|子概念|一句话介绍|
|--|--|--|
|发出|**Emit**|LLM 在回复中生成一个 tool_call 请求|
||├─ tool_call|结构化请求：`{name, arguments}`|
||└─ request|包含工具名 + JSON 参数|
|解析|**Parse**|把 LLM 输出解析成可执行的调用|
||├─ args|提取并校验参数 JSON|
||└─ JSON|处理 LLM 输出的格式错误（少括号、多字段）|
|执行|**Execute**|真正去跑这个工具|
||├─ call|调用对应函数/命令/API|
||└─ dispatch|分发到正确执行器（对应 Harness 的 Tool Runner）|

### ③ Tool Selection —— Agent 怎么选工具（工具多了怎么办）

|层级|子概念|一句话介绍|
|--|--|--|
|匹配|**Match**|从工具库里找出候选|
||├─ keyword|关键词匹配（工具名/描述里的词）|
||├─ semantic|语义匹配（向量相似度，理解意图）|
||└─ intent|意图识别（先判断用户想干什么）|
|排序|**Rank**|给候选工具打分排序|
||├─ score|相关度/置信度打分|
||└─ filter|按约束过滤（权限、环境）|
|路由|**Route**|决定最终调哪个|
||├─ top-k|选最高分那个或前几个|
||└─ fallback|没匹配到时的兜底（问用户/报错）|

### ④ Tool Types 与 Tool Ecosystem —— Agent 能调什么

|类别|子概念|一句话介绍|
|--|--|--|
|类型|**API**|通过 HTTP/REST 调外部服务|
||**Browser**|操作浏览器（打开页面、点击、填表、抓取）|
||**Code**|执行代码/命令（脚本、shell、sandbox）|
||**MCP**|Model Context Protocol：标准化的工具/资源协议|
|生态|**Registry**|工具登记与发现（对应 Harness 的 Tool Registry）|
||**MCP**|跨应用统一接入：一次实现，处处可用|
||**SDK/Plugin**|把工具封装成库/插件，方便复用与扩展|

# 第四阶段：Memory —— Agent 如何"记住

```
                                    Memory
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
  Working Memory              Episodic Memory                Semantic Memory
  （短期/工作记忆）              （情景记忆）                    （语义记忆）
        │                             │                             │
  ┌─────┼──────┐               ┌──────┼──────┐               ┌──────┼──────┐
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Scratchpad Context Stack      Event   Timeline  Session     Facts  Concepts  Embeddings
  │     │      │               │      │      │               │      │      │
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Temp   System  Call           记录    按时间  一次会话       事实   概念    向量
Results Prompt History        发生   排序    内的经历       抽取   关系    表示
  │
  ▼
  ┌───────────────┴────────────────┐
  ▼                                ▼
Long-term Memory               Memory Pipeline
  （长期记忆）                    （记忆流水线）
  │                                │
  ├─ Persistent Store ───┐         ├─ Encode ──── 写入/编码
  ├─ RAG ────────────────┤         ├─ Retrieve ── 检索/读取
  └─ Consolidation ──────┘         └─ Forget ──── 遗忘/清理
```

### ① Working Memory —— 短期工作记忆（"当前正在处理的事"）

|层级|子概念|一句话介绍|
|--|--|--|
|临时|**Scratchpad**|草稿区：存放正在计算的中间结果|
||├─ Temp Results|工具返回的临时数据，还没写回正式状态|
||└─ ...|当前步骤的中间产物|
|上下文|**Context Stack**|当前上下文栈：这一轮对话/任务看到的全部信息|
||├─ System|系统提示（角色/规则）|
||├─ Prompt|当前输入/指令|
||└─ Call History|本次任务中已发生的工具调用记录|
|特性|**易失**|会话结束/进程退出即丢失，不落盘|

### ② Episodic Memory —— 情景记忆（"发生过什么事"）

|层级|子概念|一句话介绍|
|--|--|--|
|记录|**Event**|记录发生过的事件/经历|
||├─ 记录|每次任务、每个操作的日志|
||└─ ...|包括成功/失败、当时的做法|
|组织|**Timeline**|按时间顺序组织事件|
||├─ 按时间|时间戳排序，形成时间线|
||└─ 排序|便于回溯"当时发生了什么"|
|会话|**Session**|以"一次会话"为单位的经历单元|
||├─ 一次会话|一次完整交互/任务的记录|
||└─ 内的经历|该会话里做的事、踩的坑|

### ③ Semantic Memory —— 语义记忆（"我学到了什么知识"）

|层级|子概念|一句话介绍|
|--|--|--|
|事实|**Facts**|抽取出的客观事实|
||├─ 抽取|从经历中提炼"X 是什么、Y 会导致 Z"|
||└─ ...|与具体时间/场景解耦|
|概念|**Concepts**|抽象出的概念与关系|
||├─ 概念|归纳成可复用的知识|
||└─ 关系|概念之间的关联|
|表示|**Embeddings**|用向量表示知识，便于语义检索|
||├─ 向量|把文本变成向量|
||└─ 表示|支持语义相似度匹配|

### ④ Long-term Memory —— 长期记忆（跨会话持久）

|子概念|一句话介绍|
|--|--|
|**Persistent Store**|持久化存储：数据库/文件/向量库，重启不丢|
|**RAG**|Retrieval-Augmented Generation：检索增强生成，从知识库里检索相关内容喂给 LLM|
|**Consolidation**|记忆巩固：把短期/情景记忆定期整理、提炼成长期知识|

### ⑤ Memory Pipeline —— 记忆流水线（如何运转）

|子概念|一句话介绍|
|--|--|
|**Encode**|写入：新信息编码后存入记忆库|
|**Retrieve**|读取：需要时从记忆库检索相关内容|
|**Forget**|遗忘：清理过期/无用/冲突的记忆，防止记忆膨胀|

<br/>

<br/>

# 第五阶段：Planning & Reasoning —— Agent 如何完成复杂任务

```
                          Planning & Reasoning
                                    │
      ┌──────────────┬──────────────┼──────────────┬──────────────┐
      │              │              │              │              │
      ▼              ▼              ▼              ▼              ▼
   Planning       Reasoning     Multi-step      Reflection    Replanning
      │              │            Execution         │              │
  ┌───┼───┐      ┌───┼───┐      ┌───┼───┐      ┌───┼───┐      ┌───┼───┐
  ▼   ▼   ▼      ▼   ▼   ▼      ▼   ▼   ▼      ▼   ▼   ▼      ▼   ▼   ▼
Goal Decm Sch   CoT ReAct ToT   Step State Ver  Self Crit Feed  Det  Adj  Re-ex
目标 拆解 排期   思维链 思考      单步 状态 校验   自检 批判 反馈  发现 调整 重执行



```

### ① Planning —— 先想好怎么做

|层级|子概念|一句话介绍|
|--|--|--|
|目标|**Goal**|明确最终要达成什么|
||├─ 定义目标|把模糊需求转成清晰、可验收的目标|
||└─ ...|目标要可度量（怎么算"完成"）|
|拆解|**Decompose**|把大任务拆成子任务|
||├─ 拆解|大目标 → 若干可独立执行的小步骤|
||└─ 子任务|每步有明确输入/输出|
|排期|**Schedule**|给子任务排序|
||├─ 排序/排期|决定先后顺序|
||└─ 依赖/优先级|处理依赖关系（B 依赖 A 的结果）|

### ② Reasoning —— 怎么思考

|层级|子概念|一句话介绍|
|--|--|--|
|链式|**CoT (Chain-of-Thought)**|思维链：让 LLM 一步步推演，而不是直接给答案|
||├─ 逐步推演|"先算这个，再算那个"|
||└─ ...|提升复杂逻辑/数学/推理的准确率|
|行动|**ReAct**|Reasoning + Acting：推理和行动交替进行|
||├─ 推理+行动|想一步 → 做一步 → 看结果 → 再想|
||└─ 交替进行|工具调用和思考交织（Agent Loop 的核心模式）|
|树状|**Tree-of-Thought**|思维树：同时探索多条推理路径|
||├─ 多路径探索|不只一条思路，分叉搜索|
||└─ 分支搜索|评估各分支，选最优/回溯|

### ③ Multi-step Execution —— 怎么一步步做

|层级|子概念|一句话介绍|
|--|--|--|
|步骤|**Step**|一次执行一个动作|
||├─ 单步执行|一次只做一个动作/调一个工具|
||└─ ...|避免一次做太多出错|
|状态|**State**|跟踪执行进度|
||├─ 状态更新|每步执行后更新"现在到哪了"|
||└─ ...|对应 Harness 的 State Manager|
|校验|**Verify**|每步结果是否正确|
||├─ 结果校验|检查输出是否符合预期|
||└─ ...|不对就触发重做/重规划|

### ④ Reflection —— 反思（评估做得怎么样）

|子概念|一句话介绍|
|--|--|
|**Self-check**|自我检查：做完后回顾"结果对不对、有没有遗漏"|
|**Critique**|批判：主动找自己的错误和不足|
|**Feedback**|反馈：根据错误总结经验，指导下一次改进|

### ⑤ Replanning —— 重规划（发现不对就改）

|子概念|一句话介绍|
|--|--|
|**Detect**|发现问题：执行中察觉计划行不通（工具报错、结果不符、目标变了）|
|**Adjust**|调整计划：修改步骤、换方法、重新排序|
|**Re-execute**|重新执行：按新计划继续跑|

# Multi-Agent 核心框架（第六阶段骨架）

```
                                   Multi-Agent
                                        │
          ┌─────────────────────────────┼─────────────────────────────┐
          │                             │                             │
          ▼                             ▼                             ▼
       Roles                       Delegation                  Communication
    (角色分工)                    (任务委派)                   (通信协作)
          │                             │                             │
    ┌─────┴─────┐               ┌─────┴─────┐               ┌─────┴─────┐
    ▼           ▼               ▼           ▼               ▼           ▼
Specialist  Orchestrator      Assign     Handoff         Message     Protocol
(专家角色)   (编排者)         (分配)     (交接)          (消息)      (协议)
    │           │               │           │               │           │
    ▼           ▼               ▼           ▼               ▼           ▼
 分工明确     统筹全局         Monitor     结果回传       Shared State  同步
 (分工)      (总指挥)        (监督)      (上下文)       (共享状态)   (对齐)
          ┌─────────────────────────────┼─────────────────────────────┐
          │                             │                             │
          ▼                             ▼                             ▼
   Coordination                   Supervisor                   Agent Teams
   (协同调度)                    (监督者)                     (Agent 团队)
          │                             │                             │
    ┌─────┴─────┐               ┌─────┴─────┐               ┌─────┴─────┐
    ▼           ▼               ▼           ▼               ▼           ▼
 Orchestrate   Resolve         Oversee     Escalate        定义团队    协作模式
 (编排调度)   (冲突仲裁)      (统筹监督)  (升级上报)      (编组)      (协作)
    │           │               │           │               │           │
    ▼           ▼               ▼           ▼               ▼           ▼
   同步对齐    裁决机制         汇报状态    兜底处理         角色配置    通信协议
   (Sync)     (仲裁)          (Report)   (Escalate)      (Roles)      (Comm)
   
   
   遇到任务
  │
  ├─ ① 自己能快速做？ ──────────────→ 自己做（不派）
  │
  ├─ ② 有匹配的 skill？ ────────────→ 加载 skill，按方法论做
  │
  ├─ ③ 任务重/要隔离/要专业角色？ ──→ 派 subagent
  │        └─ 派哪个？ → 描述匹配
  │        └─ 串行还是并行？ → 依赖关系
  │        └─ 结果怎么验证？ → 回传后检查
  │
  └─ ④ 都搞不定？ ──────────────────→ 升级/问用户/换策略
```

# 第七阶段：Production Agent —— 怎么把 Agent 真正做成可靠的系统

```
                              Production Agent
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
   Reliability                 Observability                Security
   （可靠性）                  （可观测性）                 （安全性）
        │                           │                           │
  ┌─────┼──────┐               ┌──────┼──────┐               ┌──────┼──────┐
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
Retry Timeout Degrade          Logs  Traces  Metrics         Auth   Sanitize  Guardrails
  │     │      │               │      │      │               │      │      │
  ▼     ▼      ▼               ▼      ▼      ▼               ▼      ▼      ▼
重试  超时   降级            日志   追踪   指标            认证   脱敏   护栏
  │
  ▼
  ┌───────────────┴────────────────┐
  ▼                                ▼
   Evaluation                  Cost & Latency
   （评估）                    （成本与延迟）
   │                                │
   ├─ Evals ──── 评估集            ├─ Token 预算
   ├─ LLM-as-Judge ── 裁判模型    ├─ 缓存
   └─ Regression ── 回归           └─ 并发控制
   │
   ▼
  ┌───────────────┴────────────────┐
  ▼                                ▼
   Deployment
   （部署）
   │
   ├─ CI/CD
   ├─ Canary
   └─ Rollback
```
