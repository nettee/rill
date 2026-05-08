---
id: "20260508-hermes-github-feishu-mvp"
name: "Hermes Github Feishu Mvp"
status: new
created: "2026-05-08"
---

## Overview

在 Hermes 原生能力上搭出一套 MVP：目标 GitHub 仓库出现新的 issue 或 PR 时，自动触发 Hermes agent 分析内容，并把分析结果发送到指定飞书群。

目标：

- 使用 Hermes 现有 webhook platform 接收 GitHub `issues` 和 `pull_request` 事件。
- 使用 Hermes Gateway 自动触发 agent 处理 webhook payload。
- 让 agent 基于 issue/PR 的 title、body、url、repo、author 等信息生成一条结构化分析消息。
- 使用 Hermes 现有 Feishu delivery 能力把 agent response 发送到指定飞书群。
- 先验证 GitHub → Hermes agent → Feishu 的端到端闭环，再决定后续是否需要 Rill 插件。


MVP 范围：

- 只覆盖 GitHub issue opened 和 PR opened 的自动通知分析。
- 只使用 Hermes 原生 webhook route、agent prompt 和 Feishu delivery 配置。
- PR 初版分析基于 webhook payload；代码 diff 级分析作为后续增强。
- 不实现 Rill 插件、持久 watcher、trigger 状态机、retry/replay 或通用事件编排层。


成功标准：

- 新 issue 创建后，飞书群收到一条 agent 生成的分析消息。
- 新 PR 创建后，飞书群收到一条 agent 生成的分析消息。
- 消息包含 repo、编号、标题、作者、链接、摘要、建议处理动作。
- webhook secret、目标 Feishu chat_id、Hermes gateway/Feishu 连接配置缺失时，系统暴露清晰失败信息。

## Research

<!-- What have we found out? What are the alternatives considered? -->

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
