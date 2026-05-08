# Hermes GitHub Issue/PR 分析到飞书群 MVP 搭建指南

目标：当目标 GitHub 仓库出现新的 issue 或 PR 时，GitHub webhook 触发本地 Hermes Gateway，Hermes agent 分析内容，并把最终分析文本发送到指定飞书群。

## 需要的组件

1. Hermes 本地配置：`/Users/william/.hermes/config.yaml`
2. Hermes Gateway：监听本地 webhook 端口 `8644`
3. Cloudflare Tunnel：把本地 `8644` 暴露为公网 HTTPS URL
4. GitHub repository webhooks：分别监听 `issues` 与 `pull_request`
5. 飞书群 `chat_id`：用于 Hermes webhook delivery 投递消息

## Hermes config.yaml 关键配置

把下面配置追加或合并到 `/Users/william/.hermes/config.yaml` 的尾部。`secret` 需要与 GitHub webhook 中配置的 secret 完全一致。

```yaml
platforms:
  webhook:
    enabled: true
    extra:
      port: 8644
      rate_limit: 30
      routes:
        github-issue-analysis:
          secret: <github_webhook_secret>
          events:
          - issues
          prompt: |
            GitHub issue webhook received. action={action}
            Only analyze when action is "opened".

            Repo: {repository.full_name}
            Issue: #{issue.number} {issue.title}
            Author: {issue.user.login}
            URL: {issue.html_url}
            Body: {issue.body}

            Return only the Chinese analysis text as your final answer.
            The webhook delivery layer will send your final answer to Feishu.
            Include: repo, number, title, author, link, summary, impact/risk, suggested owner/action, and priority.
          deliver: feishu
          deliver_extra:
            chat_id: oc_3218e07b3504dd0635bbd10fd4872cab
          actions:
          - opened
        github-pr-analysis:
          secret: <github_webhook_secret>
          events:
          - pull_request
          prompt: |
            GitHub pull request webhook received. action={action}
            Only analyze when action is "opened".

            Repo: {repository.full_name}
            PR: #{pull_request.number} {pull_request.title}
            Author: {pull_request.user.login}
            Branch: {pull_request.head.ref} -> {pull_request.base.ref}
            URL: {pull_request.html_url}
            Body: {pull_request.body}

            Return only the Chinese analysis text as your final answer.
            The webhook delivery layer will send your final answer to Feishu.
            Include: repo, number, title, author, link, summary, review focus, potential risk, suggested reviewer/action, and priority.
          deliver: feishu
          deliver_extra:
            chat_id: oc_3218e07b3504dd0635bbd10fd4872cab
          actions:
          - opened
  feishu:
    enabled: true
```

要点：

- `platforms.webhook.enabled=true` 启用 Hermes webhook platform。
- `port: 8644` 是本地 Gateway 监听端口。
- `rate_limit: 30` 控制 webhook 入口限流。
- `routes.github-issue-analysis` 处理 GitHub issue 事件。
- `routes.github-pr-analysis` 处理 GitHub PR 事件。
- `events` 过滤 GitHub 事件类型。
- `actions: [opened]` 过滤具体 action，只分析新建 issue/PR。
- `deliver: feishu` 表示 agent 最终回答由 webhook delivery 层发送到飞书。
- prompt 里需要明确要求 agent “只返回分析文本”，避免 agent 主动调用飞书工具，导致目标群只收到确认语。
- `platforms.feishu.enabled=true` 启用飞书投递能力；飞书 app 凭据沿用本机既有 Hermes 配置或登录状态。

## 启动 Hermes Gateway

修改配置后重启 Gateway：

```bash
hermes gateway restart
```

健康检查：

```bash
curl -i http://127.0.0.1:8644/health
```

期望响应：

```json
{"status":"ok","platform":"webhook"}
```

日志中应能看到两个 route 已加载：

```text
[webhook] Listening on 0.0.0.0:8644 — routes: github-issue-analysis, github-pr-analysis
```

## Cloudflare Tunnel

GitHub webhook 需要公网 HTTPS URL，本地 Hermes Gateway 需要通过 cloudflared 暴露：

```bash
cloudflared tunnel --url http://127.0.0.1:8644
```

得到公网 base URL 后，GitHub webhook payload URL 形如：

```text
https://<your-cloudflare-tunnel-host>/webhooks/github-issue-analysis
https://<your-cloudflare-tunnel-host>/webhooks/github-pr-analysis
```

本次验证使用过的 base URL：

```text
https://drinking-anne-proposition-quiz.trycloudflare.com
```

## GitHub repository webhook 配置

在目标仓库添加两个 webhook。

Issue webhook：

- Payload URL：`https://<your-cloudflare-tunnel-host>/webhooks/github-issue-analysis`
- Content type：`application/json`
- Secret：与 `github-issue-analysis.secret` 一致
- Events：`issues`
- Active：true

PR webhook：

- Payload URL：`https://<your-cloudflare-tunnel-host>/webhooks/github-pr-analysis`
- Content type：`application/json`
- Secret：与 `github-pr-analysis.secret` 一致
- Events：`pull_request`
- Active：true

GitHub repository webhook 只能按事件类型过滤，`opened` 这种 action 由 Hermes route 的 `actions: [opened]` 负责过滤。

## 验证流程

1. 确认 `http://127.0.0.1:8644/health` 返回 200。
2. 确认 cloudflared tunnel 正常代理到本地 `8644`。
3. 创建测试 issue，确认 GitHub delivery 返回 `202`，飞书群收到完整 issue 分析。
4. 创建测试 PR，确认 GitHub delivery 返回 `202`，飞书群收到完整 PR 分析。
5. 向 PR 分支继续 push，触发 `pull_request/synchronize`，确认 Hermes 返回 ignored，飞书群没有额外分析消息。

## Hermes 代码能力要求

该方案依赖 Hermes webhook adapter 支持 route-level `actions` 过滤：

- 配置：`actions: [opened]`
- 行为：payload `action` 匹配时进入 agent dispatch；不匹配时直接返回 `{"status":"ignored","event":"...","action":"..."}`。

本次验证对应的 Hermes 源码补丁：

- 本地源码：`/Users/william/projects/hermes-agent/gateway/platforms/webhook.py`
- 测试：`/Users/william/projects/hermes-agent/tests/gateway/test_webhook_adapter.py`
- 上游 PR：`https://github.com/NousResearch/hermes-agent/pull/21744`

状态说明：`actions` 过滤功能目前通过 PR 提交到上游 Hermes，PR 链接为 `https://github.com/NousResearch/hermes-agent/pull/21744`。在 PR 合入并发布前，需要使用带该补丁的本地 Hermes 运行时副本，或手动把补丁同步到当前 Hermes 安装目录。

## 常见坑

- prompt 里写 “Send to Feishu” 会让 agent 主动调用飞书工具，完整分析可能进入 Hermes home channel，目标群只收到“已发送到 Feishu 群”。prompt 应要求 agent 只返回最终分析文本。
- GitHub `pull_request` 会在后续 push 时产生 `synchronize`，需要 Hermes `actions: [opened]` 在 webhook 层过滤。
- Hermes `actions` 过滤功能目前还在上游 PR 阶段：`https://github.com/NousResearch/hermes-agent/pull/21744`。使用未包含该补丁的 Hermes 版本时，`actions: [opened]` 配置不会生效，PR `synchronize` 等 action 仍可能触发 agent 和飞书投递。
- GitHub redelivery API 可能需要额外 `admin:repo_hook` scope；测试时可以创建新的 issue/PR 触发新 delivery。
- Cloudflare Tunnel 临时 URL 会变化；URL 变化后需要同步更新 GitHub webhook Payload URL。

## 原理说明：为什么配置 `deliver`，并在 prompt 中要求 agent 只输出文本

Hermes webhook route 把一次外部 webhook 处理拆成两个阶段：

```text
GitHub webhook
→ Hermes webhook route
→ route 使用 prompt 调用 Hermes agent
→ agent 返回 final answer
→ route 的 deliver 配置把 final answer 投递到目标平台
```

因此 `prompt` 与 `deliver` 分别负责两件事：

- `prompt`：定义 agent 如何理解 GitHub payload，以及最终生成什么分析内容。
- `deliver` / `deliver_extra`：定义 agent 的最终回答发送到哪里，例如发送到飞书的哪个 `chat_id`。

当前配置里：

```yaml
prompt: |
  Return only the Chinese analysis text as your final answer.
  The webhook delivery layer will send your final answer to Feishu.
deliver: feishu
deliver_extra:
  chat_id: oc_3218e07b3504dd0635bbd10fd4872cab
```

含义是：agent 只负责产出中文分析正文；Hermes webhook delivery 层负责把这段正文发送到指定飞书群。

这样设计的核心价值是职责清晰：

- agent 负责认知任务：总结 issue/PR、判断风险、给出建议动作。
- Hermes 平台层负责工程任务：验签、路由、限流、投递、记录日志。
- 飞书目标群由配置控制，行为可审计、可复现。

prompt 中需要明确要求 agent “只输出分析文本”，因为 `deliver: feishu` 已经承担发送动作。如果 prompt 写成 “Send to Feishu”，agent 可能会主动调用飞书工具，随后 webhook delivery 又会把 agent 的最终确认语发送到目标群，导致目标群只收到类似“已发送到 Feishu 群”的确认文本。正确分工是：agent 输出正文，delivery 发送正文。

## Agent 智能判断通知的后续扩展方向

当前 MVP 使用固定投递模式：route 命中、agent 生成 final answer 后，`deliver: feishu` 会把 final answer 发到飞书。`actions: [opened]` 负责在 agent dispatch 前过滤，只让新建 issue/PR 进入分析与投递流程。

如果希望 agent 根据事件内容智能判断是否通知飞书群，可以考虑以下扩展方式。

### 方式一：去掉 `actions`，保留固定 `deliver`

适用于希望所有相关事件都进入 agent，由 agent 决定消息轻重，飞书群仍然保留一条消息记录的场景。

```yaml
events:
- pull_request
prompt: |
  GitHub pull request webhook received. action={action}

  Decide whether this event needs human attention.
  Return only the Chinese message that should be sent to Feishu.
deliver: feishu
deliver_extra:
  chat_id: oc_3218e07b3504dd0635bbd10fd4872cab
```

在这种模式下，`opened`、`synchronize`、`ready_for_review` 等 action 都会触发 agent，且 final answer 都会发送到飞书。agent 可以输出“需要重点 review”的分析，也可以输出“本次同步风险较低”的低噪声说明。

### 方式二：移除固定 `deliver`，让 agent 主动决定是否调用飞书工具

适用于 MVP 试验“agent 自主决定是否发群”的场景。

配置思路是删除：

```yaml
deliver: feishu
deliver_extra:
  chat_id: ...
```

并在 prompt 中写清楚判断规则与目标群：

```text
Decide whether this event should notify the Feishu group.

Send a Feishu message only when the event needs human attention.
Use target chat_id: oc_3218e07b3504dd0635bbd10fd4872cab.

If no notification is needed, return a concise internal note only.
```

这种方式把“是否发送”交给 agent 的工具调用决策。它适合快速验证智能判断效果，但稳定性依赖 agent 是否严格遵循 prompt，并且需要确保 agent 使用正确的目标 `chat_id`。

### 方式三：增加 conditional delivery 能力

适用于希望保留配置式 delivery，同时让 agent 智能决定是否发送的生产化方案。该方式需要扩展 Hermes 代码。

理想行为是让 agent 返回结构化结果：

```json
{
  "notify": true,
  "message": "需要发送到飞书群的中文分析文本"
}
```

或者：

```json
{
  "notify": false,
  "reason": "PR synchronize 只是同步提交，无需通知"
}
```

然后 Hermes delivery 层在发送前解析该结构：

```text
notify=true  → 发送 message 到飞书
notify=false → 跳过飞书发送，并记录 skipped / ignored
```

可能的配置形态：

```yaml
deliver: feishu
deliver_extra:
  chat_id: oc_3218e07b3504dd0635bbd10fd4872cab
  conditional: true
  notify_field: notify
  message_field: message
```

该扩展需要覆盖以下行为：

- `notify=true` 时发送 `message`。
- `notify=false` 时跳过发送。
- JSON 无效时显式失败，避免误发或静默吞错。
- 缺少必需字段时显式失败。
- 默认保持现有 `deliver: feishu` 固定投递行为。

推荐演进路径：MVP 使用 `actions: [opened]` 固定过滤；需要智能判断时先用方式二验证 prompt 与 agent 决策效果；验证稳定后，把方式三做成 Hermes 的 conditional delivery 能力。
