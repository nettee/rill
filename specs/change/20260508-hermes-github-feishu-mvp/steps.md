## Step 1: Agent 只读盘点当前仓库与 GitHub 远端

状态：完成。

- 1.1 当前 git remote：`origin git@github.com:nettee/rill.git`，当前分支 `main`，跟踪 `origin/main`。
- 1.2 GitHub repo：`nettee/rill`，默认分支 `main`，仓库 URL `https://github.com/nettee/rill`，SSH URL `git@github.com:nettee/rill.git`。
- 1.2 GitHub CLI：已登录账号 `nettee`，token scopes 包含 `repo`，当前仓库权限 `ADMIN`，`viewerCanAdminister=true`；可读取 repo hooks，当前 hooks 列表为空。
- 1.3 当前工作区：仅有本文件 `specs/change/20260508-hermes-github-feishu-mvp/steps.md` 为未跟踪文件。测试 PR 安全变更策略：后续使用专用测试分支，只修改该 spec 目录下的记录文件。
- 1.4 当前 open issues：无。当前 open PR：无。测试标题无冲突。
- 1.5 记录：repo=`nettee/rill`，默认分支=`main`，测试分支建议=`hermes-mvp-test-20260508`，测试 issue 标题=`[Hermes MVP Test] issue opened`，测试 PR 标题=`[Hermes MVP Test] PR opened`。

偏差/坑：`gh repo view --json issuesEnabled` 字段名无效，改用 `hasIssuesEnabled` 后读取成功。
