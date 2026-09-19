# 怎么读 pi：抓住设计哲学

这是一份个人阅读笔记，不是仓库文档。目标：用最少代码理解 pi **为什么长成这样**。TUI、主题、按键、CLI 边角、provider 适配细节都可以先跳过。

作者原话在两处：

- 产品哲学：`packages/coding-agent/README.md` 的 **Philosophy**，以及 [这篇博客](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- 运行时哲学：`packages/agent/docs/harness.md` 的 Part 0

---

## 1. 先记住一句话

pi 不是「功能最多的 coding agent」。它是一个 **最小、可自扩展的 agent harness**。

核心故意不做的事，比它做了什么更能说明设计：

| 故意不做 | 替代方式 |
|---|---|
| 不内置 MCP | 写 CLI + skill README，或自己做 extension |
| 不内置 sub-agent | tmux 再开一个 pi，或自己做 extension |
| 不内置权限弹窗 | 容器 / sandbox，或自己做确认流 |
| 不内置 plan mode | 写文件，或自己做 |
| 不内置 todo 工具 | `TODO.md`，作者认为内置 todo 会把模型搞糊涂 |
| 不内置 background bash | 用 tmux |

产品口号是：**Adapt pi to your workflows, not the other way around, without having to fork internals.**

读代码时，每看到一个「看起来缺功能」的地方，先问：这是不是故意留空，让 extension / skill / package 去填？

---

## 2. 仓库在干什么

这是 monorepo，不是单一 CLI。分层从底到顶：

```text
packages/ai              统一 LLM API（只有会 tool calling 的模型）
packages/agent           agent runtime：loop、会话、持久化、harness
packages/coding-agent    把 runtime 装成 coding agent（工具、session、扩展、CLI）
packages/chord           插件 / 多进程组合 runtime（experimental 在用）
packages/protocol        experimental 远程协议
packages/tui             终端 UI  ← 先跳过
```

读设计时只需要前两层，再加 coding-agent 的 **core**，不要从 CLI 入口开始。

当前有 **两套 agent 运行时**，这是读仓库最容易晕的地方：

1. **经典 loop（现在的主路径）**  
   `Agent` + `agentLoop`。进程内、事件驱动、JSONL session。  
   coding-agent 的 SDK / 交互模式走这条：`createAgentSession()` → `new Agent()`。

2. **durable harness（正在成为规范实现）**  
   `AgentHarness` + `driveOperation`。会话可崩溃恢复，状态机驱动。  
   spec 在 `packages/agent/docs/harness.md`。experimental session worker 走这条。

先把经典 loop 读透，再读 harness。不要反过来：harness 文档很长，会把「agent 是什么」淹没在存储不变量里。

---

## 3. 贯穿全仓库的设计原则

下面这些原则在代码里反复出现。读任何文件时用它们当滤镜。

### 3.1 核心要小，行为用组合填

coding-agent 默认只给模型四个工具：`read` / `write` / `edit` / `bash`。系统提示也很短。作者的判断是：前沿模型已经被 RL 训成 coding agent，不需要 1 万 token 的系统提示。

扩展点是一等公民：

- **Skills**：给模型看的说明书（按需 `read` 技能文件）
- **Extensions**：TypeScript 模块，能注册工具、拦截事件、改 compaction
- **Prompt templates**：用户侧斜杠命令
- **Pi packages**：把上面这些打包分享

「自己问 pi 做一个 extension」是预期用法，不是客套。

### 3.2 精确控制模型看到什么

博客里写得很直白：**context engineering is paramount**。现有 harness 的通病是在背后塞东西进 context，UI 还看不见。

pi 的缝专门为这件事设计：

```text
AgentMessage[]          应用层 transcript（可以有 UI-only / 自定义消息）
        │ transformContext()     裁剪、注入、compaction 准备
        ▼
AgentMessage[]
        │ convertToLlm()         过滤、把自定义类型变成 LLM 认识的四种 role
        ▼
Message[]               真正发给模型的东西
```

对应代码：

- 类型：`packages/agent/src/types.ts`（`AgentMessage`、`convertToLlm`、`transformContext`）
- 转换发生点：`packages/agent/src/agent-loop.ts` 的 `streamAssistantResponse()`
- coding-agent 的自定义消息：`packages/agent/src/harness/messages.ts`（`bashExecution`、`compactionSummary`、`branchSummary`、`custom`）

读任何「消息 / session / compaction」代码，都先问：**这段是改存储，还是改模型看到的 context？** 两者在 pi 里是分开的。compaction 不删历史，只改下次发给模型的窗口。

### 3.3 LLM 内容和 UI 内容拆开

工具返回值有两份：

- `content`：给模型
- `details`：给日志 / UI，不进模型 context

见 `AgentToolResult`（`packages/agent/src/types.ts`）。这是故意的：不要为了好看的 TUI 把结构化数据塞进 prompt。

### 3.4 Loop 本身几乎没有策略

`agentLoop` **没有 max steps**。它一直转到模型不再 call tool、也没有 queued follow-up。作者的理由：没用过这个旋钮，就不加。

策略全在回调里：

| 回调 | 时机 | 典型用途 |
|---|---|---|
| `beforeToolCall` / `afterToolCall` | 每个 tool | 拦截、改写结果 |
| `shouldStopAfterTurn` | 一轮结束后 | 优雅停 |
| `prepareNextTurn` | 下一轮 LLM 调用前 | compaction、换模型 |
| `getSteeringMessages` | tool 跑完后 | 用户中途插话 |
| `getFollowUpMessages` | agent 本可以停时 | 排队的下一条 |

所以「agent 怎么想」不在 loop 里。loop 只负责：发请求 → 跑 tool → 再发请求。怎么裁 context、怎么停、怎么插入用户消息，都是宿主的事。`Agent` 类只是给这些回调加上状态和队列。

### 3.5 失败走协议，不靠抛异常穿过 loop

`StreamFn` 的契约（`types.ts`）：请求失败必须编码进返回的 stream（`stopReason: "error" | "aborted"`），不能 throw。`convertToLlm` / `transformContext` 同样约定不能 throw。

原因：loop 是事件序列。中途扔异常会让 UI / session 落在半截事件上。读 loop 时把 `stopReason` 当控制流，不要按普通 async 函数想。

### 3.6 Session 是树，不是线性日志

会话 JSONL 是带 `parentId` 的树。当前工作点是一条 branch 的 tip。`/tree` 跳历史、`/fork` 开新会话、compaction 追加 summary entry，都建立在这棵树上。

存储上：历史 append-only，不删。compaction 改的是 **发给模型的窗口**，不是磁盘上的记录。这和 harness spec 里「deletion is not a runtime feature」是同一条原则。

### 3.7 持久化 runtime 的四条不变量

如果继续读 `AgentHarness`，整份 spec 都从这四条推出来（`harness.md` §0.3）：

1. **三个 store，没有第四个地方**  
   entries（对话树，写一次） / values·lists（可变当前值） / usage ledger（费用）。任何该留下的东西必须落在其中一个。
2. **原子事务**  
   一次 commit 要么全成，要么看不见。
3. **完整当前状态是重启点**  
   `operationState` 每次替换成完整快照，不靠 replay journal，也不靠「缺了什么」来猜进度。
4. **intent → 不确定的外部效果 → settlement**  
   LLM 请求和 tool 调用都先提交「我即将做 X」，再做，再提交结果。崩溃发生在效果窗口里时，不假装 exactly-once。

Harness **明确不保证**：外部效果 exactly-once、接回 provider 的半截 stream、多写者、调度、复制。这些 non-goals 比 features 更重要。

工具还有一条顺序原则（`docs/tool-durability.md`）：并行 tool **完成顺序** 和写进对话的 **source order** 不是一回事。完成了先耐久化到 `pending.entry`，按 assistant 声明顺序再进树。否则崩溃会把已经跑完的 tool 当成没跑。

---

## 4. 推荐阅读顺序

按这个顺序读，每一步只回答一个问题。读完可以停；后面是加厚，不是主线。

### 第 0 天：先建立口味（约 30 分钟）

1. `README.md`（仓库根）— 包地图。
2. `packages/coding-agent/README.md` 的开头 + **Philosophy** 整节。
3. [博客原文](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/) 里这些小节：
   - *Context handoff*
   - *Structured split tool results*
   - *Minimal agent scaffold*
   - *Minimal system prompt / toolset*
   - *YOLO by default*
   - *No built-in to-dos / plan / MCP / background bash / sub-agents*

读到这里你应该能用自己的话解释：pi 为什么拒绝成为 Claude Code。

### 第 1 天：经典 agent loop（这是核心）

只读这几个文件，尽量读全文：

1. `packages/agent/src/types.ts`  
   先搞清：`StreamFn`、`AgentMessage`、`AgentLoopConfig`、`AgentTool` / `AgentToolResult`。  
   特别看 `convertToLlm` 和 `CustomAgentMessages` 的注释。

2. `packages/agent/src/agent-loop.ts`  
   从 `agentLoop` → `runAgentLoop` → `runLoop` 往下。内层 while 是 tool/steering，外层 while 是 follow-up。  
   然后读 `streamAssistantResponse`（context 变换边界）和 `executeToolCalls`（校验发生在 loop，不在 provider）。

3. `packages/agent/src/agent.ts` 的 `Agent` 类  
   只是 loop 的状态壳：transcript、steering/follow-up 队列、把 `prompt()` 转成一次 `runAgentLoop`。  
   `createLoopConfig()` 能看清所有策略回调从哪进来。

4. `packages/agent/src/harness/messages.ts` 的 `convertToLlm()`  
   看自定义 role 怎么变成 `user` 消息。这是「应用 transcript ≠ 模型 context」的实例。

读完应能在纸上画出一次 `prompt("读 README")` 的事件序列：`agent_start` → `turn_start` → user message → assistant（可能带 toolCall）→ tool 执行 → toolResult → 下一 turn → `agent_end`。

`packages/agent/README.md` 的 Event Flow 图就是这份作业的标准答案。

### 第 2 天：coding-agent 怎么把 loop 变成产品

仍然跳过 TUI。只看 core：

1. `packages/coding-agent/src/core/sdk.ts`  
   `createAgentSession()`：装工具、extensions、把 `Agent` 嵌进去。  
   注意 `setDefaultStreamFn(streamSimple)` 的注释：agent-core **故意不依赖** 具体 provider。

2. `packages/coding-agent/src/core/agent-session.ts` 开头注释 + `prepareNextTurn` / compaction 相关方法  
   这是所有 mode（interactive / print / rpc）共享的宿主。它把 session 树、compaction、模型切换接到 `Agent` 的回调上。文件很长，不要通读；搜 `prepareNextTurn`、`compact`、`steer`、`followUp`。

3. `packages/coding-agent/src/core/system-prompt.ts`  
   系统提示怎么从 preamble + 工具 snippet + AGENTS.md + skills 拼起来。对照「最小 prompt」原则。

4. `packages/coding-agent/src/core/tools/read.ts`（一个工具就够）  
   看 `content` vs `details`、`operations` 可替换（本地 fs / SSH）。工具是数据 + 执行，不是和 TUI 焊死的。

5. `packages/coding-agent/docs/extensions.md` 开头 + Events 总览  
   产品哲学落地的地方：权限、plan mode、sub-agent 都应该能在这里做，而不是改 loop。

6. `packages/coding-agent/docs/sessions.md` + `docs/compaction.md` 的 Overview  
   树、fork、compaction 不删历史。

到这里，**pi 作为 coding agent 的设计已经闭环**：小 loop + 小工具集 + 把策略放到 session/extension 层。

### 第 3 天：LLM 边界（只读接口，不读每个 provider）

`packages/ai` 很大，90% 是各家 API 的脏细节。设计上只需要：

1. `packages/ai/README.md` 开头到 Tools / Context Serialization / Cross-Provider Handoffs。
2. 搞清：模型必须能 tool call；`streamSimple` 满足 `StreamFn`；abort 必须返回 partial；跨 provider 切换是 best-effort（thinking 变成带标签的文本）。
3. 不要读 `openai-completions.ts` 这类文件，除非你要修某个 provider。

作者自己写过：统一 API 必然 leaky，所以 pi-ai 直接包各家 SDK，而不是再套一层 Vercel AI SDK。

### 第 4 天：durable harness（只有你关心「崩溃后续跑」时再读）

现在的 CLI 主路径还不是这套。读它是为了理解 **pi 想把 agent 变成什么系统**，不是为了改当前交互模式。

只读 spec 的 Part 0，不要一开始啃 1400 行全文：

1. `packages/agent/docs/harness.md` §0.1–0.6  
   系统模型、三个 store、Slack 例子、崩溃在 tool 中间的例子、non-goals。
2. `packages/agent/docs/runtime-simplification.md` 的 Core model + Durable state  
   13 个 `at` 叶子；可见顺序永远是 `prepare → publish intent → perform effect → publish outcome`。  
   作者明确禁止再引入 generic Procedure / scheduler / graph。
3. `packages/agent/src/harness/runtime/drive.ts` 的 `driveOperation()`  
   就是那张状态表的 dispatcher。每个 `case` 点进对应 `drive/*.ts` 即可，不必一次读完。
4. `packages/agent/docs/work-packages/06-session-branch-lane-separation.md` 开头的四概念：

   ```text
   Session       全局耐久数据 + 一条 mutation line
   Branch        对话树上的一条路径，tip 可移动
   AgentLane     Branch + 配置 + 至多一个 operation
   AgentHarness  管 lanes，自己不是 lane
   ```

5. 需要时再读 `docs/tool-durability.md`、`docs/assistant-durability.md`。它们回答的是同一句话：外部效果不可靠，所以用 intent/settlement 和 source-order materialization 把不确定性关进明确的窗口。

**不要读：** `pico.md` / `pico2.md` / `pico-v3.md`。那是讨论中的下一版设计，不是当前代码。文件自己写了 *Design under discussion*。

---

## 5. 一张图：一次用户输入走哪

经典路径（你现在实际跑 `pi` 时）：

```text
用户 prompt
  → AgentSession.prompt()
  → Agent.prompt()
  → runAgentLoop()
       transformContext / convertToLlm
       streamFn(model, llmContext)          ← packages/ai
       executeToolCalls()                   ← 校验 + before/after hooks
       prepareNextTurn()                    ← compaction 常挂在这
       getSteeringMessages / getFollowUpMessages
  → 事件流回 AgentSession
  → SessionManager 把消息追加到 JSONL 树
```

Harness 路径（experimental / 未来）：

```text
lane.prompt()
  → accept（耐久创建 operation，还没有效果）
  → driveOperation()
       starting → checkpoint → assistant.ready
       → 提交 intent → 调模型 → settlement 写入 entry
       → tools：每个 call 同样 intent → effect → outcome_ready → 按 source order 进树
  → 进程死了：读完整 operationState，从所在叶子继续
```

两套共享同一套消息/工具类型，但 **控制权存放处不同**：前者在进程内存 + 回调，后者在 Session 的 values 里。

---

## 6. 明确跳过

这些对设计哲学几乎没有增量，会拖慢你：

- `packages/tui/**`、interactive components、themes、keybindings
- `packages/ai/src/api/*` 各 provider 实现
- `packages/ai/src/models.generated.ts`（生成文件）
- coding-agent 的 HTML export、Windows/Termux 文档
- harness 的 work-package 全文、telemetry schema、JSONL codec 边角
- `packages/chord` 除非你要读 experimental 多进程 UI
- `pico*` 设计稿

遇到 extension 例子时，看 `examples/extensions/plan-mode` 或 `subagent` 就够了：它们证明「核心不做，是因为能在外面做」。

---

## 7. 读完你应该能回答

如果答不上，回到对应文件，不要继续往下堆细节：

1. 为什么默认只有四个工具、而且没有权限系统？
2. `AgentMessage` 和发给模型的 `Message` 为什么是两套？compaction 改的是哪一套？
3. `agentLoop` 在什么条件下停？谁有权让它停？
4. 工具的 `content` 和 `details` 分别给谁看？
5. 经典 `Agent` 和 `AgentHarness` 各自把「当前跑到哪」存在哪里？崩溃后谁能续、谁不能？
6. 一个新功能（比如 plan mode）按 pi 的口味应该改 core，还是写 extension？

能答这六个，设计哲学就读够了。之后再按具体问题钻文件。
