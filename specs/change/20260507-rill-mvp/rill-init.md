# Rill Init

## Synclo 做了什么

`/Users/william/projects/synclo`

Synclo 是 OpenClaw 的事件驱动协调插件。

它的核心能力：

- 通过 `synclo_watch` 创建 watcher，订阅外部事件。
- 通过 `synclo_task` 跟踪跨 session / 跨人的任务。
- 通过 `synclo_update` 更新 watcher / task 状态。
- 通过 `synclo_search` 查找已有 watcher / task，避免重复创建。
- 接收 GitHub、Feishu、deadline 等外部事件。
- 将事件写入事件表，匹配 watcher，生成 trigger。
- 在 agent 处理前注入 pending trigger、watcher instructions、历史上下文。
- 唤醒 OpenClaw agent / subagent 处理事件。


Synclo 的价值是让 agent 拥有跨 session 的事件记忆和事件触发能力。

## 在 Hermes 中实现 Rill 插件

希望模仿 synclo，在 Hermes Agent 中实现类似能力的插件，名叫 Rill。

Rill 要实现的核心功能：

- 接收外部事件
- 在事件命中时唤醒 Hermes agent。
- 为 agent 注入事件相关上下文。
- agent 结果能发送到 Feishu channel。


Hermes 中的对应能力：

- Hermes plugin：注册 Rill tools。
- Hermes Gateway：统一接收外部 channel / webhook / cron / API 事件。
- Hermes Feishu adapter：统一发送到 Feishu DM 或群聊。
- `pre_llm_call` hook：在 agent 处理前注入 Rill 上下文。
- `send_message` / Gateway delivery：将结果发送到指定 channel。
