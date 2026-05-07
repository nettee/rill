---
id: 20260507-rill-mvp
name: Rill MVP
status: researched
created: '2026-05-07'
---

## Overview

希望模仿 synclo，在 Hermes Agent 中实现类似能力的插件，名叫 Rill。

Synclo 是 OpenClaw 的事件驱动协调插件。代码：`/Users/william/projects/synclo`

Rill 要实现的核心功能：

- 接收外部事件
- 在事件命中时唤醒 Hermes agent。
- 为 agent 注入事件相关上下文。
- agent 结果能发送到 Feishu channel。

MVP 阶段实现能力：

- 目标 GitHub 仓库出现新的 issue 或 PR 时，分析 issue 或 PR 的内容，发送一条消息到飞书群

## Research

### Existing System

- Rill 当前处于规划阶段；仓库里的 active spec 只定义了目标能力，尚无 `src` 实现代码。Source: `specs/change/20260507-rill-mvp/spec.md:8-19`
- Rill 目标是模仿 Synclo，在 Hermes Agent 中实现插件能力：接收外部事件、事件命中时唤醒 Hermes agent、注入事件上下文、把 agent 结果发送到 Feishu channel。Source: `specs/change/20260507-rill-mvp/spec.md:10-19`
- 现有需求草稿把 Hermes 映射为：plugin 注册 Rill tools，Gateway 统一接收 channel/webhook/cron/API 事件，Feishu adapter 发送 DM 或群聊，`pre_llm_call` hook 在 agent 处理前注入 Rill 上下文，`send_message` / Gateway delivery 发送结果。Source: `specs/change/20260507-rill-mvp/rill-init.md:35-41`
- Hermes 插件以目录形式提供 `plugin.yaml` 和 `__init__.py`，`register(ctx)` 负责注册工具、hook、命令等能力；用户插件可放在 `~/.hermes/plugins/`，项目插件可放在 `.hermes/plugins/` 并通过 `HERMES_ENABLE_PROJECT_PLUGINS=true` 启用。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:21-31,45-90,92`
- Hermes 插件 API 支持 `ctx.register_tool`、`ctx.register_hook`、`ctx.register_command`、`ctx.dispatch_tool`、`ctx.inject_message`、`ctx.register_platform` 等扩展点。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:94-113`
- Hermes 插件发现来源包括 bundled、user、project、pip、Nix；general plugins 使用 `plugins.enabled` allow-list 启用，bundled platform/backend 等基础设施按类别自动加载。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:116-126,143-180`
- Hermes 的 `PluginContext.register_tool` 把插件工具注册到全局 `tools.registry`，并记录为 plugin-provided tool，因此 Rill tools 会和内置工具走同一个 registry/dispatch 路径。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:237-273`
- Hermes 的 `PluginContext.register_hook` 把 callback 存入 `PluginManager._hooks`；未知 hook 会 warning 后继续保存，用于前向兼容。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:532-547`
- Hermes 的 `invoke_hook` 会调用所有已注册 callback，单个 callback 异常会记录 warning，返回所有非空结果；`pre_llm_call` 返回的 `{"context": ...}` 或字符串会作为当前 turn 的用户消息上下文。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:1089-1123,1197-1202`
- Hermes 文档定义 `pre_llm_call` 每个 user turn 触发一次，是当前唯一使用返回值的 hook，可把 context 注入当前 user message；注入内容是 ephemeral，不写入 session DB，多插件结果按 discovery 顺序用空行合并。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/hooks.md:504-543`
- Hermes `run_agent.py` 在 tool-calling loop 前调用 `pre_llm_call`，传入 `session_id`、`user_message`、`conversation_history`、`is_first_turn`、`model`、`platform`、`sender_id`，再把插件上下文追加到当前 user message。Source: `/Users/william/projects/hermes-agent/run_agent.py:10884-10918,11099-11120,11144-11156`
- Hermes Gateway 的消息流是 platform adapter 接收 raw event，规范化为 `MessageEvent`，`GatewayRunner._handle_message()` 解析 session key、授权、slash command、running agent，随后创建 `AIAgent` 并运行 conversation；session key 格式为 `agent:main:{platform}:{chat_type}:{chat_id}`，构造应使用 `build_session_key()`。Source: `/Users/william/projects/hermes-agent/website/docs/developer-guide/gateway-internals.md:52-78`
- Hermes Gateway `_handle_message` 在用户消息进入授权前触发 `pre_gateway_dispatch` 插件 hook，插件可返回 `skip`、`rewrite`、`allow` 来影响 gateway dispatch。Source: `/Users/william/projects/hermes-agent/gateway/run.py:4856-4914`
- Hermes Gateway 创建或复用缓存的 `AIAgent`，把 `platform`、`user_id`、`chat_id`、`chat_type`、`thread_id`、`gateway_session_key` 等来源信息传入 agent，再调用 `run_conversation`。Source: `/Users/william/projects/hermes-agent/gateway/run.py:13690-13755,14059-14094`
- Hermes Gateway hook 系统通过 `~/.hermes/hooks/<name>/HOOK.yaml` 和 `handler.py` 发现事件 hook，支持 `gateway:startup`、`session:start`、`agent:start`、`agent:step`、`agent:end`、`command:*` 等事件，handler 异常会记录并继续主流程。Source: `/Users/william/projects/hermes-agent/gateway/hooks.py:1-20,64-81,158-181`
- Hermes 的 `send_message` tool 支持 `send` 和 `list`，target 形如 `platform`、`platform:#channel-name`、`platform:chat_id`、`platform:chat_id:thread_id`；发送前会解析 channel directory 和 gateway config。Source: `/Users/william/projects/hermes-agent/tools/send_message_tool.py:117-145,148-221`
- `send_message` tool 通过 `_check_send_message` 要求 gateway 正在运行或当前 session 在 messaging platform；工具注册在 `tools.registry` 的 `messaging` toolset 下。Source: `/Users/william/projects/hermes-agent/tools/send_message_tool.py:1664-1674,1778-1788`
- Hermes DeliveryRouter 能解析 `origin`、`local`、`platform`、`platform:chat_id`、`platform:chat_id:thread_id` 目标，并调用对应 platform adapter delivery。Source: `/Users/william/projects/hermes-agent/gateway/delivery.py:1-9,28-47,50-95,129-169`
- Hermes Feishu adapter 的 outbound `send` 支持 `chat_id`、content、reply target 和 metadata；会构造 post/text payload，post payload 被 Feishu 拒绝时 fallback 到 plain text，并返回 `SendResult`。Source: `/Users/william/projects/hermes-agent/gateway/platforms/feishu.py:1693-1752`
- Hermes 的插件级 `ctx.inject_message` 只支持 CLI active conversation；gateway mode 下没有 CLI reference 时返回 false，因此 Rill 在 Gateway 里唤醒 agent 应走 Gateway message/session 路径。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:275-301`
- Synclo 的插件入口使用 `definePluginEntry`，注册 `synclo_watch`、`synclo_task`、`synclo_update`、`synclo_search` 等工具，并在注册时初始化数据库。Source: `/Users/william/projects/synclo/src/plugin/index.ts:360-444,446-570`
- `synclo_watch` 创建持久 watcher：写入 `content`、订阅事件列表 `events`、上下文、owner、deadline、agent/session 来源。Source: `/Users/william/projects/synclo/src/plugin/index.ts:402-430`
- Synclo 在 `before_prompt_build` hook 中根据 agent identity 收集上下文，把系统指导和动态 context 注入 prompt。Source: `/Users/william/projects/synclo/src/plugin/index.ts:780-837`
- Synclo 在 `message_received` hook 中维护 session 到 identity 的映射。Source: `/Users/william/projects/synclo/src/plugin/index.ts:896-909`
- Synclo 的 `gatherContext` 会原子领取 pending triggers，并返回 `pending_triggers`、`active_nodes`、`channel_nodes`。Source: `/Users/william/projects/synclo/src/core/context.ts:38-50,54-83,136-175`
- Synclo 的事件匹配分三类：精确匹配 `node_watches.watch_source`、分段通配符匹配、语义匹配。Source: `/Users/william/projects/synclo/src/core/match-nodes.ts:29-33,48-105,119-145,167-175`
- Synclo 的 `EventProcessor` 轮询 `events` 表，将 pending 事件 claim 为 processing，匹配 active nodes，创建 trigger，并把事件标记 handled；失败事件会按 retry/backoff 回到 pending 或 ignored。Source: `/Users/william/projects/synclo/src/daemon/event-processor.ts:24-64,87-103`
- Synclo 的 `createTrigger` 会把 node、trigger、事件 payload、历史 trigger 写入 `injected_context`，供后续 agent 处理。Source: `/Users/william/projects/synclo/src/core/create-trigger.ts:56-105`
- Synclo 的 GitHub webhook 只接受 `POST /webhook`，支持 direct mode 签名校验，解析 GitHub event/delivery 后先返回 200，再处理 webhook 事件并触发 batch/dispatch。Source: `/Users/william/projects/synclo/src/sources/github/webhook.ts:201-243,255-270`
- GitHub webhook 会把 issues、issue_comment、pull_request、review 等 GitHub payload 规范化为 `github:<repo>:...` 事件类型。Source: `/Users/william/projects/synclo/src/sources/github/webhook.ts:295-371,375-381`
- Synclo 的 GitHub handler 会先运行 EventProcessor，再读取 pending triggers，按 sessionKey 串行、不同 sessionKey 并行地调用 Gateway 触发 agent。Source: `/Users/william/projects/synclo/src/sources/github/handler.ts:451-482,524-540,945-947`
- Synclo dispatch 失败时会把 trigger 重置为 pending，让后续 retry 周期继续处理。Source: `/Users/william/projects/synclo/src/sources/github/handler.ts:510-522,924-934`
- Synclo 的 Gateway client 支持两种触发 agent 模式：插件模式通过 `api.runtime.subagent.run()`，standalone 模式通过 OpenClaw CLI fallback。Source: `/Users/william/projects/synclo/src/sources/github/gateway.ts:1-7`
- Synclo 的 Feishu provider 通过 app id/secret 获取 tenant token，能列出群聊和群成员；当前代码表现为目录解析能力。Source: `/Users/william/projects/synclo/src/directory/providers/feishu.ts:3-11,13-31,38-76`
- Synclo 集成测试覆盖完整生命周期：agent 创建 watcher，外部 Feishu 事件写入，EventProcessor 创建 pending trigger，下一次 prompt build 注入 trigger，上下文处理后 agent 标记 done。Source: `/Users/william/projects/synclo/test/integration-openclaw.test.ts:79-160`

### Available Approaches

- **Synclo-style 持久事件表 + watcher/trigger 表**：外部事件先进入事件队列，处理器匹配 watcher 并创建 trigger，prompt hook 再注入 trigger。Source: `/Users/william/projects/synclo/src/daemon/event-processor.ts:24-64,87-103`; `/Users/william/projects/synclo/src/plugin/index.ts:780-837`
- **Gateway 直接派发 agent**：事件处理后通过 Gateway runtime/CLI 触发 agent session，适合 Hermes Gateway 已承担统一事件与 delivery 的架构。Source: `/Users/william/projects/synclo/src/sources/github/handler.ts:924-934`; `/Users/william/projects/synclo/src/sources/github/gateway.ts:1-7`; `specs/change/20260507-rill-mvp/rill-init.md:37-41`
- **精确/通配事件命名协议**：沿用 `provider:scope:entity:id:action` 风格事件名，可支持精确订阅和通配订阅。Source: `/Users/william/projects/synclo/src/core/match-nodes.ts:48-55,67-90,167-175`; `/Users/william/projects/synclo/src/sources/github/webhook.ts:301-371`
- **Hook 注入上下文**：在 Hermes 的 `pre_llm_call` 中注入 Rill 上下文，对应 Synclo 的 `before_prompt_build` 注入点。Source: `specs/change/20260507-rill-mvp/rill-init.md:37-40`; `/Users/william/projects/synclo/src/plugin/index.ts:780-837`
- **Hermes plugin-native Rill**：把 Rill 做成 Hermes general plugin，通过 `plugin.yaml` + `register(ctx)` 注册 `rill_watch` / `rill_update` / `rill_search`、`pre_llm_call` 和必要的 gateway hook。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:21-31,94-113`; `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:237-273,532-547`
- **Gateway ingress hook / adapter 入口**：Rill 可先用 `pre_gateway_dispatch` 获取消息事件并做 skip/rewrite/allow，也可通过独立 webhook/cron/API ingress 写入 Rill 事件队列，再复用 Hermes session key 触发 agent。Source: `/Users/william/projects/hermes-agent/gateway/run.py:4856-4914`; `/Users/william/projects/hermes-agent/website/docs/developer-guide/gateway-internals.md:52-78`
- **Hermes 发送路径**：Rill 结果可由 agent 调用 `send_message`，也可由 Rill 后台使用 DeliveryRouter/Feishu adapter 直接 delivery；前者复用 agent 权限和工具，后者适合后台系统消息。Source: `/Users/william/projects/hermes-agent/tools/send_message_tool.py:117-145,1664-1788`; `/Users/william/projects/hermes-agent/gateway/delivery.py:129-169`; `/Users/william/projects/hermes-agent/gateway/platforms/feishu.py:1693-1752`
- **Agent 自行发送 Feishu**：Synclo 的 GitHub handler 把 notify target 放进 agent 任务消息，让 agent 使用对应 channel tool 发送；Rill 可改用 Hermes Feishu adapter / Gateway delivery 统一发送。Source: `/Users/william/projects/synclo/src/sources/github/handler.ts:833-857`; `specs/change/20260507-rill-mvp/rill-init.md:39-41`

### Constraints & Dependencies

- Rill 依赖 Hermes 已存在或将提供的 plugin tools、Gateway event ingress、Feishu adapter、`pre_llm_call` hook、`send_message` / delivery API。Source: `specs/change/20260507-rill-mvp/rill-init.md:35-41`
- Hermes 的 `pre_llm_call` context 注入到 user message，并且是 ephemeral；Rill 的 pending trigger claim/状态推进需要自己持久化，prompt 注入本身不会修改 session DB。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/hooks.md:526-543`; `/Users/william/projects/hermes-agent/run_agent.py:11099-11120`
- Hermes 当前可确认的 Gateway agent 入口是消息进入 `_handle_message` 后创建/复用 `AIAgent` 并调用 `run_conversation`；Rill 需要在设计阶段确定后台 trigger 是伪造/投递 `MessageEvent`、调用 Gateway 内部方法，还是创建一条独立 one-shot agent path。Source: `/Users/william/projects/hermes-agent/gateway/run.py:4856-4867,13690-13755,14059-14094`
- Hermes 插件 `ctx.inject_message` 在 gateway mode 返回 false，适合作为 CLI 注入能力，后台事件唤醒 Hermes Gateway agent 需要使用 Gateway/session 相关路径。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:275-301`
- Hermes user/project general plugins 受 `plugins.enabled` 控制；Rill 安装后需要明确启用方式、配置位置和必要 env。Source: `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:143-164`
- Hermes 的 plugin hook callback 异常会被捕获并记录 warning；Rill 的关键事件处理需要在自己的队列/processor 中持久化失败状态，保障可观测失败和 retry。Source: `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:1089-1123`; `/Users/william/projects/hermes-agent/gateway/hooks.py:158-181`
- Feishu 发送可走 `send_message` 或 Feishu adapter；两条路径都返回结构化 success/error，Rill 需要把失败结果写入 trigger/delivery 状态。Source: `/Users/william/projects/hermes-agent/tools/send_message_tool.py:167-221,1664-1788`; `/Users/william/projects/hermes-agent/gateway/platforms/feishu.py:1697-1752`
- 当前 Rill 仓库没有实现代码可复用，下一阶段需要确认 Hermes 内部 API 的稳定性边界、Rill 是否调用 Gateway private method、以及是否新增公开 wake/dispatch API。Source: `specs/change/20260507-rill-mvp/spec.md:8-19`; `/Users/william/projects/hermes-agent/gateway/run.py:4856-4867`
- Synclo 的实现强依赖数据库状态机：`events.status`、`triggers.status`、session identity、node/watch 表；Rill 也需要等价的持久化或明确使用 Hermes 提供的持久层。Source: `/Users/william/projects/synclo/src/daemon/event-processor.ts:24-64`; `/Users/william/projects/synclo/src/core/context.ts:62-83`
- 事件接收路径需要幂等、claim、retry/backoff、dispatch failure 重新 pending，避免事件丢失或重复处理。Source: `/Users/william/projects/synclo/src/daemon/event-processor.ts:34-63`; `/Users/william/projects/synclo/src/sources/github/handler.ts:501-522`
- Feishu 能力需要 app id/secret/token，并区分目录解析与消息发送；Synclo 现有 Feishu provider 只展示了 token、群列表、成员列表能力。Source: `/Users/william/projects/synclo/src/directory/providers/feishu.ts:3-11,13-31,38-76`
- Prompt 注入要有 claim 语义；Synclo 使用事务把 pending trigger 更新为 injected，防止并发 prompt 领取同一个 trigger。Source: `/Users/william/projects/synclo/src/core/context.ts:54-83`

### Key References

- `specs/change/20260507-rill-mvp/spec.md:8-19` - Rill MVP 目标能力。
- `specs/change/20260507-rill-mvp/rill-init.md:35-41` - Hermes 侧预期集成点。
- `/Users/william/projects/hermes-agent/website/docs/user-guide/features/plugins.md:21-180` - Hermes 插件结构、能力、发现来源与启用规则。
- `/Users/william/projects/hermes-agent/hermes_cli/plugins.py:237-301,532-547,1089-1123,1197-1202` - PluginContext 工具/hook/message injection API 与 hook invocation 行为。
- `/Users/william/projects/hermes-agent/website/docs/user-guide/features/hooks.md:504-543` - `pre_llm_call` 语义、参数、返回值与 ephemeral 注入规则。
- `/Users/william/projects/hermes-agent/run_agent.py:10884-10918,11099-11120` - `pre_llm_call` 在 agent loop 中的调用与 user message 注入实现。
- `/Users/william/projects/hermes-agent/website/docs/developer-guide/gateway-internals.md:52-78` - Gateway 消息流与 session key 格式。
- `/Users/william/projects/hermes-agent/gateway/run.py:4856-4914,13690-13755,14059-14094` - Gateway dispatch hook、AIAgent 创建/复用与 conversation 调用。
- `/Users/william/projects/hermes-agent/gateway/hooks.py:1-20,64-81,158-181` - Gateway lifecycle hook 系统。
- `/Users/william/projects/hermes-agent/tools/send_message_tool.py:117-221,1664-1788` - `send_message` tool schema、target 解析、可用性 gate 与注册。
- `/Users/william/projects/hermes-agent/gateway/delivery.py:1-9,28-95,129-169` - Delivery target parsing 与 delivery router。
- `/Users/william/projects/hermes-agent/gateway/platforms/feishu.py:1693-1752` - Feishu adapter outbound send 行为。
- `/Users/william/projects/synclo/src/plugin/index.ts:360-444,780-837,896-909` - Synclo 插件入口、工具注册、prompt hook、session identity hook。
- `/Users/william/projects/synclo/src/core/context.ts:38-83,136-175` - pending trigger 原子领取与上下文结构。
- `/Users/william/projects/synclo/src/core/match-nodes.ts:29-145,167-175` - 事件到 watcher 的精确/通配/语义匹配。
- `/Users/william/projects/synclo/src/daemon/event-processor.ts:24-103` - 事件队列处理与 trigger 创建。
- `/Users/william/projects/synclo/src/core/create-trigger.ts:56-105` - injected_context 数据形状。
- `/Users/william/projects/synclo/src/sources/github/webhook.ts:201-270,295-371` - webhook ingress 与事件规范化。
- `/Users/william/projects/synclo/src/sources/github/handler.ts:451-540,833-857,924-947` - trigger dispatch、Feishu notify 指令、agent 唤醒。
- `/Users/william/projects/synclo/test/integration-openclaw.test.ts:79-160` - watch → event → trigger → context injection → done 的集成测试样例。

## Design

<!-- Technical approach, architecture decisions, and test strategy. Each design decision should cite a fact source. -->

## Plan

<!-- Optional: Step breakdown for complex features that need multiple implementation steps.
     Decided during Design. Checked off during Implement.
     Keep this section compact and step-based.
     Use markdown checkboxes for all step and substep items, for example:
     - [ ] Step 1: Foo
       - [ ] Substep 1.1 Implement: Foo foundation
       - [ ] Substep 1.2 Implement: Foo integration
       - [ ] Substep 1.3 Implement: Foo edge handling
       - [ ] Substep 1.4 Verify: Foo automated coverage
       - [ ] Substep 1.5 Verify: Foo manual workflow
     - [ ] Step 2: Bar
       - [ ] Substep 2.1 Implement: Bar
       - [ ] Substep 2.2 Verify: Bar
     - [ ] Step 3: Baz
       - [ ] Substep 3.1 Implement: Baz
       - [ ] Substep 3.2 Verify: Baz
     Use a capability-based step breakdown with reviewable, meaningful increments.
     Good boundaries align with one user-visible workflow, one subsystem/integration boundary, one migration/rollout step, or one stabilization milestone.
     Each step must include small, independent substeps for implementation and immediate testing/verification.
     Within each step, list implementation substeps before verification substeps.
     The final step may focus on overall testing/verification, edge cases, regression coverage, and coverage improvements.
     A step is complete only when relevant tests pass.
     Size steps so one coding agent can implement + validate in a single session.
     Write each substep as one small, independent task. -->

## Notes

<!-- Optional sections — add what's relevant. -->

### Implementation

<!-- Files created/modified, decisions made during coding, deviations from design -->

### Verification

<!-- How the feature was verified: tests written, manual testing steps, results -->
