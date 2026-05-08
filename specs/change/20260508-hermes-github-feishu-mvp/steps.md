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
