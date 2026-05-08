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

状态：完成初次确认，并完成 prompt 修复与 retry 验证；等待人工确认 retry 消息是否已按预期进入目标飞书群。

- 6.1 人工确认目标飞书群 `oc_3218e07b3504dd0635bbd10fd4872cab` 收到初次 issue 通知，但内容只有 `已发送到 Feishu 群。`。
- 6.2 人工确认完整 GitHub Issue 分析消息进入了此前配置的 Hermes home channel。完整消息包含 repo、编号、标题、作者、链接、摘要、影响/风险、建议动作、优先级。
- 6.3 原因分析：route prompt 使用了 `Send a concise Chinese analysis...`，agent 将其理解为主动调用 Feishu 发送工具；该工具默认投递到 Hermes home channel。webhook route 的 `deliver=feishu` 随后把 agent 最终回复 `已发送到 Feishu 群。` 投递到了目标群，导致目标群只收到摘要/确认语。
- 6.3 修复：已修改 `/Users/william/.hermes/config.yaml` 中 `github-issue-analysis` 与 `github-pr-analysis` prompts，明确要求 `Return only the Chinese analysis text as your final answer. The webhook delivery layer will send your final answer to Feishu.`。
- 6.3 已执行 `hermes gateway restart`，`/health` 返回 200 OK；日志确认 `[webhook] Listening on 0.0.0.0:8644 — routes: github-issue-analysis, github-pr-analysis`。
- 6.3 Retry 验证：GitHub redelivery API 返回 404，提示当前 token 需要 `admin:repo_hook` scope；改为创建 retry issue `https://github.com/nettee/rill/issues/2`，标题 `[Hermes MVP Test] issue opened retry`。
- 6.3 Retry delivery：hook id `619555132`，delivery id `3818704905724494000`，event/action=`issues/opened`，status=`OK`，status_code=`202`，delivered_at=`2026-05-08T06:50:42.255Z`。
- 6.3 Hermes retry 日志：`[webhook] POST event=issues route=github-issue-analysis prompt_len=585 delivery=3a55218a-4aaa-11f1-85d5-0511f8845a90`；agent response ready，response=`651 chars`，随后 webhook delivery 发送该 651 字符响应。
- 6.4 已在 retry issue 添加验证评论：`https://github.com/nettee/rill/issues/2#issuecomment-4404201849`。

偏差/坑：Step 6 发现了目标群只收到 agent 确认语、完整分析进入 home channel 的路由问题；根因是 prompt 让 agent 主动发送 Feishu，与 webhook delivery 重叠。修复后使用 retry issue 代替 GitHub redelivery，因为 redelivery API 需要额外 token scope。
