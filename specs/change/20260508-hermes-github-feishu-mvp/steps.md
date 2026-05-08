## Step 1: Agent 只读盘点当前仓库与 GitHub 远端

状态：完成。

- 1.1 当前 git remote：`origin git@github.com:nettee/rill.git`，当前分支 `main`，跟踪 `origin/main`。
- 1.2 GitHub repo：`nettee/rill`，默认分支 `main`，仓库 URL `https://github.com/nettee/rill`，SSH URL `git@github.com:nettee/rill.git`。
- 1.2 GitHub CLI：已登录账号 `nettee`，token scopes 包含 `repo`，当前仓库权限 `ADMIN`，`viewerCanAdminister=true`；可读取 repo hooks，当前 hooks 列表为空。
- 1.3 当前工作区：仅有本文件 `specs/change/20260508-hermes-github-feishu-mvp/steps.md` 为未跟踪文件。测试 PR 安全变更策略：后续使用专用测试分支，只修改该 spec 目录下的记录文件。
- 1.4 当前 open issues：无。当前 open PR：无。测试标题无冲突。
- 1.5 记录：repo=`nettee/rill`，默认分支=`main`，测试分支建议=`hermes-mvp-test-20260508`，测试 issue 标题=`[Hermes MVP Test] issue opened`，测试 PR 标题=`[Hermes MVP Test] PR opened`。

偏差/坑：`gh repo view --json issuesEnabled` 字段名无效，改用 `hasIssuesEnabled` 后读取成功。

## Step 2: Agent 只读检查 Hermes 本地状态

状态：完成。

- 2.1 Hermes config：存在 `/Users/william/.hermes/config.yaml`，大小 9581 bytes；`/Users/william/.config/hermes/` 不存在。可见顶层配置键包含 `agent`、`providers`、`model`、`slack`、`discord`、`telegram`、`mattermost`、`whatsapp` 等。
- 2.2 Webhook platform：当前配置未见 `platforms` 节，未见 `platforms.webhook`；因此 webhook enabled、port、rate limit、routes 均未配置。按 spec 需要新增 `github-issue-analysis` 和 `github-pr-analysis` 两个 route。
- 2.3 Feishu platform：当前配置未见 `platforms.feishu`，也未发现包含 `feishu` 的键；未见 Feishu `chat_id` 或 home channel。按 spec 需要配置 Feishu 凭据与目标群 `chat_id`。
- 2.4 Hermes webhook health：访问 `http://127.0.0.1:8644/health` 失败，错误为无法连接到 `127.0.0.1:8644`，说明本地 8644 端口当前没有 webhook 服务响应。
- 2.5 人工配置缺口：需要人工修改 Hermes 本地配置，启用 webhook platform，设置端口/rate limit/routes/secret/prompt/Feishu delivery/chat_id，填入 Feishu 凭据和 GitHub webhook secret，并启动或重启本地 Hermes Gateway。

偏差/坑：本机 Python 环境缺少 `yaml` 模块，改用 Ruby 标准库 `YAML` 做只读配置摘要，避免输出敏感值。

## Step 3: 人工配置 Hermes webhook 与 Feishu delivery

状态：完成。

- 3.1 已修改 `/Users/william/.hermes/config.yaml`，新增 `platforms.webhook.enabled=true`，端口 `8644`，`rate_limit=30`。
- 3.2 已配置 `github-issue-analysis` route：生成并写入 route secret，`events=[issues]`，issue prompt，`deliver=feishu`，`deliver_extra.chat_id` 使用 `oc_3218e07b3504dd0635bbd10fd4872cab`。
- 3.3 已配置 `github-pr-analysis` route：使用同一个 route secret，`events=[pull_request]`，PR prompt，`deliver=feishu`，`deliver_extra.chat_id` 使用同一个飞书群。
- 3.4 已设置 `platforms.feishu.enabled=true`。当前只写入目标群 `chat_id`；Feishu app 凭据需依赖用户本机现有 Hermes Feishu 登录/配置。后续启动时若凭据缺失，Hermes/Feishu adapter 应暴露连接失败。
- 3.5 人工已执行 `hermes gateway restart`，Hermes Gateway 已重启。
- 3.6 `/health` 验证通过：`HTTP/1.1 200 OK`，body 为 `{"status": "ok", "platform": "webhook"}`。
- 3.7 Hermes 日志确认 route 已加载：`[webhook] Listening on 0.0.0.0:8644 — routes: github-issue-analysis, github-pr-analysis`。公开 webhook base URL：`https://drinking-anne-proposition-quiz.trycloudflare.com`。

偏差/坑：用户要求 agent 直接修改 Hermes config，原计划中 Step 3 的配置写入从人工操作调整为 agent 操作。第一次 Ruby 写入脚本因 prompt 中 `#{issue.number}` 被 Ruby 字符串插值解析而失败，发生在写文件前；改用 `%q{}` 原样字符串后写入成功。验证摘要显示两个 route 均有 secret、chat_id、正确 event 和 Feishu delivery。

## Step 4: Agent 自动配置当前仓库 GitHub webhooks

状态：完成。

- 4.1 已创建 issue route webhook，hook id `619555132`，Payload URL：`https://drinking-anne-proposition-quiz.trycloudflare.com/webhooks/github-issue-analysis`。
- 4.2 issue webhook：content type 为 `json`，secret 使用 Hermes route 中同一个 secret，事件为 `issues`，active=true。
- 4.3 已创建 PR route webhook，hook id `619555148`，Payload URL：`https://drinking-anne-proposition-quiz.trycloudflare.com/webhooks/github-pr-analysis`。
- 4.4 PR webhook：content type 为 `json`，secret 使用 Hermes route 中同一个 secret，事件为 `pull_request`，active=true。
- 4.5 GitHub hook 列表验证通过：两个 webhook URL、content type、event、active 状态均正确；GitHub last_response 均为 active/200 OK。
- 4.6 GitHub 自动 ping delivery 验证通过：issue hook delivery id `3818703399015153700`，event=`ping`，status=`OK`，status_code=200；PR hook delivery id `3818703403299635000`，event=`ping`，status=`OK`，status_code=200。

偏差/坑：第一次创建脚本缺少 Ruby `shellwords` 依赖，命令在调用 GitHub API 前失败，未创建 webhook；补充 `require "shellwords"` 后创建成功。

## Step 5: Agent 自动触发 issue webhook 验证

状态：完成，等待 Step 6 人工确认飞书群消息内容。

- 5.1 已创建测试 issue：`https://github.com/nettee/rill/issues/1`，标题 `[Hermes MVP Test] issue opened`，作者 `nettee`，创建时间 `2026-05-08T06:39:56Z`。
- 5.2 GitHub issue webhook recent delivery 确认收到 `issues` / `opened`。
- 5.3 Delivery HTTP response：hook id `619555132`，delivery id `3818703523336421400`，status=`OK`，status_code=`202`，delivered_at=`2026-05-08T06:39:58.56Z`。
- 5.3 Hermes Gateway 日志确认收到事件：`[webhook] POST event=issues route=github-issue-analysis prompt_len=540 delivery=babc459e-4aa8-11f1-8c01-8d9fc41fa120`；随后 agent response ready，platform=`webhook`，time=`33.2s`。
- 5.4 已在测试 issue 添加验证评论：`https://github.com/nettee/rill/issues/1#issuecomment-4404123453`。

偏差/坑：`gh issue create --json` 在当前 GitHub CLI 版本中不可用，改为创建后用 `gh issue view --json` 读取 issue 元数据。

## Step 6: 人工确认 issue 飞书消息

状态：完成。

- 6.1 人工确认目标飞书群 `oc_3218e07b3504dd0635bbd10fd4872cab` 收到初次 issue 通知，但内容只有 `已发送到 Feishu 群。`。
- 6.2 人工确认完整 GitHub Issue 分析消息进入了此前配置的 Hermes home channel。完整消息包含 repo、编号、标题、作者、链接、摘要、影响/风险、建议动作、优先级。
- 6.3 原因分析：route prompt 使用了 `Send a concise Chinese analysis...`，agent 将其理解为主动调用 Feishu 发送工具；该工具默认投递到 Hermes home channel。webhook route 的 `deliver=feishu` 随后把 agent 最终回复 `已发送到 Feishu 群。` 投递到了目标群，导致目标群只收到摘要/确认语。
- 6.3 修复：已修改 `/Users/william/.hermes/config.yaml` 中 `github-issue-analysis` 与 `github-pr-analysis` prompts，明确要求 `Return only the Chinese analysis text as your final answer. The webhook delivery layer will send your final answer to Feishu.`。
- 6.3 已执行 `hermes gateway restart`，`/health` 返回 200 OK；日志确认 `[webhook] Listening on 0.0.0.0:8644 — routes: github-issue-analysis, github-pr-analysis`。
- 6.3 Retry 验证：GitHub redelivery API 返回 404，提示当前 token 需要 `admin:repo_hook` scope；改为创建 retry issue `https://github.com/nettee/rill/issues/2`，标题 `[Hermes MVP Test] issue opened retry`。
- 6.3 Retry delivery：hook id `619555132`，delivery id `3818704905724494000`，event/action=`issues/opened`，status=`OK`，status_code=`202`，delivered_at=`2026-05-08T06:50:42.255Z`。
- 6.3 Hermes retry 日志：`[webhook] POST event=issues route=github-issue-analysis prompt_len=585 delivery=3a55218a-4aaa-11f1-85d5-0511f8845a90`；agent response ready，response=`651 chars`，随后 webhook delivery 发送该 651 字符响应。
- 6.4 已在 retry issue 添加验证评论：`https://github.com/nettee/rill/issues/2#issuecomment-4404201849`。
- 6.4 人工确认 retry 消息已正常进入目标飞书群 `[H] Rill 测试`，内容为完整 issue 分析文本，包含 repo、编号、标题、作者、链接、摘要、影响/风险、建议动作等信息。

偏差/坑：Step 6 发现了目标群只收到 agent 确认语、完整分析进入 home channel 的路由问题；根因是 prompt 让 agent 主动发送 Feishu，与 webhook delivery 重叠。修复后使用 retry issue 代替 GitHub redelivery，因为 redelivery API 需要额外 token scope。

## Step 7: Agent 自动触发 PR webhook 验证

状态：完成，等待 Step 8 人工确认飞书群消息内容。

- 7.1 已基于当前本地 `main` 创建测试分支 `hermes-mvp-test-20260508`。本节记录变更同时作为无害文档/spec 变更，用于打开测试 PR。
- 7.1 已提交并推送测试分支，提交 `27bfac5 step: start pr webhook verification`。
- 7.2 已打开测试 PR：`https://github.com/nettee/rill/pull/3`，标题 `[Hermes MVP Test] PR opened`，head=`hermes-mvp-test-20260508`，base=`main`，作者 `nettee`，创建时间 `2026-05-08T07:03:46Z`。
- 7.3 GitHub PR webhook recent delivery 确认收到 `pull_request` / `opened`。
- 7.4 Delivery HTTP response：hook id `619555148`，delivery id `3818706595582312400`，status=`OK`，status_code=`202`，delivered_at=`2026-05-08T07:03:49.158Z`。
- 7.4 Hermes Gateway 日志确认收到事件：`[webhook] POST event=pull_request route=github-pr-analysis prompt_len=736 delivery=0f3b4950-4aac-11f1-8cc9-2e93f8556bda`；随后 agent response ready，response=`852 chars`，webhook delivery 已发送该响应。
- 7.5 已在测试 PR 添加验证评论：`https://github.com/nettee/rill/pull/3#issuecomment-4404324243`。

偏差/坑：由于当前本地 `main` 已有多步记录提交且未推送，测试分支从本地 `main` 创建并推送，PR 会包含这些步骤记录提交。该行为符合当前 spec 验证需要，也让远端 PR 有实际变更可触发 webhook。

## Step 8: 人工确认 PR 飞书消息

状态：完成，并修复非 opened PR action 额外通知问题。

- 8.1 人工确认目标飞书群 `[H] Rill 测试` 收到 `[Hermes MVP Test] PR opened` 对应完整 PR 分析消息。
- 8.2 消息包含 repo、PR 编号、标题、作者、链接、摘要、review focus、风险/建议动作等信息。
- 8.3 异常 1：飞书群额外收到一条 `synchronize` 消息，内容为“本次 webhook 事件为 synchronize，按要求仅在 opened 时进行分析”。原因是 Step 7 打开 PR 后又 push 了记录提交，GitHub 对 PR webhook 发送 `pull_request/synchronize`；Hermes 原本只支持 event type 过滤，仍会触发 agent 并通过 Feishu delivery 投递 no-op 说明。
- 8.3 异常 2：PR 验证评论最初包含字面量 `\n`，原因是 `gh pr comment` 命令中换行转义方式错误。已用 GitHub API 更新 comment `4404324243` 为正常多行内容。
- 8.3 修复：在 Hermes webhook adapter 增加 route-level `actions` 过滤，位置：`/Users/william/projects/hermes-agent/gateway/platforms/webhook.py` 与运行时副本 `/Users/william/.hermes/hermes-agent/gateway/platforms/webhook.py`。当 route 配置 `actions: [opened]` 时，payload `action` 不匹配会在 agent dispatch 前返回 `{"status":"ignored","event":"...","action":"..."}`，避免 no-op 消息投递到 Feishu。
- 8.3 测试：在 Hermes 源码仓库运行 `/Users/william/.hermes/hermes-agent/venv/bin/python -m pytest tests/gateway/test_webhook_adapter.py -q`，结果 `38 passed`。
- 8.3 配置：已在 `/Users/william/.hermes/config.yaml` 的 `github-issue-analysis` 与 `github-pr-analysis` routes 添加 `actions: [opened]`，并执行 `hermes gateway restart`，`/health` 返回 200 OK。
- 8.3 Smoke verify：手工向本地 `github-pr-analysis` route 发送签名正确的 `pull_request/synchronize` payload，返回 `HTTP/1.1 200 OK` 与 `{"status":"ignored","event":"pull_request","action":"synchronize"}`，确认不会触发 agent。
- 8.3 GitHub verify：修复后再次 push Step 8 记录触发 `pull_request/synchronize` delivery id `3818708026712391700`，GitHub 记录 status=`OK`，status_code=`200`；此前未修复的 synchronize delivery id `3818706805033304000` 为 status_code=`202`。新的 200 表示 webhook 已在 action filter 处同步忽略，未进入 agent accepted 流程。
- 8.4 人工确认 PR 飞书消息主体正常；额外 `synchronize` 消息的根因已修复，后续非 opened action 将在 webhook 层忽略。

偏差/坑：为满足 MVP “只覆盖 opened” 标准，本步骤从纯配置调整为 Hermes webhook adapter 小幅增强；GitHub repository webhook 无法按 `action` 过滤，只能按 `pull_request` 事件类型过滤。

## Step 9: Agent 自动整理验证记录与清理 GitHub 测试资源

状态：完成。

- 9.1 已更新 spec Notes，记录 repo、webhook IDs、测试 issue/PR URL、delivery IDs、人工 Feishu 确认结果、Hermes 配置需求与 Hermes 源码补丁。
- 9.1 Hermes 需要的本地配置主要在 `/Users/william/.hermes/config.yaml`：`platforms.webhook.enabled=true`、`platforms.webhook.extra.port=8644`、`rate_limit=30`、两个 routes、route secrets、`actions: [opened]`、prompts、`deliver=feishu`、`deliver_extra.chat_id=oc_3218e07b3504dd0635bbd10fd4872cab`、`platforms.feishu.enabled=true`。Feishu app 凭据沿用用户本机既有 Hermes 配置/登录状态。
- 9.1 Hermes 源码也有变更：`/Users/william/projects/hermes-agent/gateway/platforms/webhook.py` 增加 route-level `actions` 过滤，`/Users/william/projects/hermes-agent/tests/gateway/test_webhook_adapter.py` 增加 action filter 测试；Hermes 提交 `a4349304c step: add webhook action filtering`。运行时副本 `/Users/william/.hermes/hermes-agent/gateway/platforms/webhook.py` 同步应用了该补丁。
- 9.1 已通过 fork 向上游 Hermes 创建 PR：`https://github.com/NousResearch/hermes-agent/pull/21744`，标题 `fix(webhook): filter route actions before agent dispatch`，分支 `nettee:fix-webhook-action-filter`，提交 `a790501b fix(webhook): filter route actions before agent dispatch`。PR 全程使用英文并按 upstream PR template 填写；包含源码、测试和文档三类变更，目标测试 `/Users/william/.hermes/hermes-agent/venv/bin/python -m pytest tests/gateway/test_webhook_adapter.py -q` 结果 `56 passed`。
- 9.2 已关闭测试 issue #1：`https://github.com/nettee/rill/issues/1`，并添加清理评论。
- 9.2 已关闭 retry issue #2：`https://github.com/nettee/rill/issues/2`，并添加清理评论。
- 9.3 已关闭测试 PR #3：`https://github.com/nettee/rill/pull/3`，并添加清理评论。
- 9.3 已删除远端测试分支 `origin/hermes-mvp-test-20260508`。
- 9.4 验证：issue #1、issue #2、PR #3 状态均为 `CLOSED`。
- 9.5 验证：GitHub webhooks 保留 active：issue hook `619555132`，PR hook `619555148`；两个 hook last_response 均为 active/200 OK。

偏差/坑：Step 9 记录提交在本地测试分支上完成；远端测试分支已按清理计划删除，因此本地记录提交用于审计，最终需要按项目需要决定是否 cherry-pick/merge 到 `main`。
