# 多智能体编排（对照 OpenMAIC prototype）

本文只描述当前 prototype 的课堂讨论编排，对应实现在 `lib/orchestration/`。生成流水线（大纲 / 场景）不在此范围。

## 目标

一次用户发言或一次客户端续轮，只跑 **一轮**：导演决定下一个说话者，至多让一个智能体生成回复，然后图结束。多轮讨论由客户端把上一轮返回的 `directorState` 和消息历史再提交一次来串起来。图本身不循环，也没有 `maxTurns`。

## 图

```
START → director ──(shouldEnd)──→ END
              │
              └──(next)──→ agent_generate → END
```

入口：`createOrchestrationGraph()` / `buildInitialState()`（`lib/orchestration/director-graph.ts`），由 `lib/orchestration/stateless-generate.ts` 调用。节点通过 LangGraph custom stream 的 `config.writer()` 推聊天事件（`thinking`、`cue_user`、`agent_start`、`text_delta`、`agent_end`），供 SSE 下发。一次发言就是一段口播文本。

## 状态

`OrchestratorState` 在请求进入时写入，节点只改可变字段。

| 字段 | 来源 | 用途 |
|---|---|---|
| `messages` | 请求 | 对话历史 |
| `storeState` | 请求 | 当前场景、白板是否打开 |
| `availableAgentIds` | `config.agentIds` | 本轮可选智能体 |
| `agentConfigOverrides` | `config.agentConfigs` | 生成型智能体，随请求携带，不写入服务端注册表 |
| `discussionContext` | `discussionTopic` / `discussionPrompt` | 有则进入讨论模式 |
| `triggerAgentId` | `config.triggerAgentId` | 第 0 轮快路径发言人 |
| `userProfile` | 请求 | 学生昵称与背景 |
| `turnCount` / `agentResponses` | `directorState` | 客户端带回的跨请求进度 |
| `currentAgentId` / `shouldEnd` | 导演节点 | 路由 |

智能体解析顺序：请求级 override，然后全局 `useAgentRegistry`。

## 导演

`directorNode` 按人数分支，避免无谓的模型调用。

**单智能体（代码，不调模型）**

- `turnCount === 0`：派发唯一智能体。
- 之后：写 `cue_user`，`shouldEnd = true`。

**多智能体**

1. `turnCount === 0` 且 `triggerAgentId` 在可选列表中：直接派发该智能体。
2. 否则调模型。系统提示由 `buildDirectorPrompt` 拼出（模板 `lib/prompts/templates/director/system.md`），输入包括智能体名单（id、name、role、priority）、已发言摘要、对话压缩、学生档案。
3. `parseDirectorDecision` 得到下一发言人：
   - `END` 或空 id：结束。
   - `USER`：`cue_user` 后结束。
   - 未知 id：结束。
   - 合法 id：写入 `currentAgentId`，进入 `agent_generate`。
4. 模型异常：结束。

讨论模式与问答模式只改提示规则：讨论时发起者先说，教师再引导，其他学生补充；问答时优先教师。

## 智能体生成

`agent_generate` 只处理导演选出的一个 id，产出是一段聊天发言。

1. `buildStructuredPrompt` 组装该角色的系统提示（人格、同伴本轮发言、学生档案）。角色细则与篇幅目标仍在 `prompt-builder.ts`。
2. 历史经 `convertMessagesToOpenAI` 映射：其他智能体的发言对当前智能体视为 user，避免模型把自己和别人混成同一条 assistant。
3. 消息列表必须以 `HumanMessage` 结尾；否则补一条开场或“轮到你发言”的提示。
4. 流式文本写入 `text_delta`。本轮摘要（谁说了什么）随 `AgentTurnSummary` 返回，供下一请求的导演和同伴上下文使用。

结构化 action 不是这条编排的主体。OpenMAIC 代码里还留着白板、聚光灯等类型，prototype 的对话不使用那一套，文档不把它们算进动作面。

## 角色

`AgentConfig`（`lib/orchestration/registry/types.ts`）描述一个聊天角色：`role`、`persona`、`priority`（1–10，导演排序用）、可选 TTS。

## 和客户端的契约

服务端无会话。客户端每次请求带上：

- 消息历史
- `config.agentIds`（以及生成型智能体的 `agentConfigs`）
- 课堂 `storeState`
- 上一轮的 `directorState`（`turnCount`、`agentResponses`）

一轮结束后，客户端根据事件更新 UI，若导演没有 `cue_user` 或结束，再发下一轮。

## 文件索引

| 路径 | 职责 |
|---|---|
| `lib/orchestration/director-graph.ts` | 图、导演节点、单轮生成 |
| `lib/orchestration/director-prompt.ts` | 导演提示与决策解析 |
| `lib/orchestration/prompt-builder.ts` | 智能体结构化提示 |
| `lib/orchestration/stateless-generate.ts` | 请求入口与流式解析 |
| `lib/orchestration/registry/` | 智能体配置与选择 |
| `lib/orchestration/summarizers/` | 对话压缩、同伴上下文 |
| `lib/prompts/templates/director/` | 导演 markdown 模板 |
| `lib/prompts/templates/agent-system*` | 智能体系统提示模板 |
