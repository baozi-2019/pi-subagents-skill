---
name: swarm
description: |
  多子代理群（swarm）编排：把一个复杂任务分解为有边界的多条子代理通道，
  通过 pi-subagents 的 subagent 工具并行/分阶段派发、回收并汇总结果。
  当用户输入 /swarm <任务>、或明确要求"用多个子代理/并行派发/swarm 方式"完成任务时使用。
---

# Swarm：多子代理自动编排

用户输入 `/swarm <任务>` 即构成**明确的委派授权**（满足 pi-subagents 的
operator-requested delegation 门槛）。本 skill 是父代理（你）的编排手册：
你始终是编排者，负责分解、派发、仲裁、验收与交付；子代理只做有边界的执行。

开始编排前，若对 `subagent` 工具的字段或 `runs.run` / `runs.all` / `runs.lanes`
用法不确定，先读取 pi-subagents skill 的参考文档：

- 角色与提示词：`skills/pi-subagents/references/prompting-and-roles.md`
- 执行控制（async、workflowScript 等）：`skills/pi-subagents/references/execution-controls.md`

路径相对于 pi-subagents 包目录（通常为
`~/.pi/agent/npm/node_modules/pi-subagents/`）。也可调用
`subagent({action:"guide",topic:"workflows"})` 获取权威用法。

## 编排流程

### 1. 理解任务与定界

- 复述任务目标（一句话），明确交付物形态（代码改动、调研报告、审查结论等）。
- 确认工作边界：目标仓库/目录、允许修改的范围、只读还是可写。
- 任务边界不清、存在多种合理解释时，**先用 `ask_user_question` 澄清**，不要带着歧义派发。

### 2. 分解为通道（lane）

按"接缝"而非"编号"分解——每条通道有独立的证据来源或修改范围：

| 任务形态 | 推荐形状 |
| --- | --- |
| 小而聚焦，一个子代理即可 | 直接 `subagent({agent, task})`，不要为形式造并行 |
| 多个相互独立的子任务（不同模块/文件/调研角度） | 一个 `workflowScript`，`runs.all([...])` 并行扇出 |
| 有先后依赖（先侦察/设计，后实现/审查） | `workflowScript` 中按序 `await runs.run(...)`，或 `runs.lanes(...)` |
| 实现 + 独立验收 | 先 writer，完成后 fresh-context reviewer 审查，父代理汇总修复 |

规则：
- **不制造并行**：仅当子代理能带来独立证据、专门化、有用并行或隔离性时才拆分；2~4 条通道是常见上限。
- **一个写者一个目录**：多个可写子代理不得共用同一 cwd/worktree；有重叠修改风险时用 `worktree: true` 隔离，或串行。
- 通道提示词必须有区分度，禁止只换序号的克隆提示词。

### 3. 派发

- 默认 **async**（异步后台）；只有父代理必须阻塞等待结果时才 `async:false`。
- 每个 `runs.run` / `runs.all` 条目给出 `label`（短动词短语，如 `label: "调研鉴权模块"`）与稳定的 `key`。
- 每个子代理任务包必须**冷启动完备**，逐项写清：
  1. 目标（一句话）
  2. 仓库 / cwd / ref
  3. 权限边界（可写范围或只读）
  4. 相关文件、契约、约束
  5. 成功 / 验收标准
  6. 验证方式（构建、测试命令）
  7. 期望输出 / 汇报格式
  8. 停止与求助条件
- 工作者/侦察走快而够用的模型档，正式审查走强模型档；父代理保持默认强模型。

### 4. 回收与汇总

- 异步通道有原生完成通知；启动或分诊完有用的异步通道后让出控制权，等 Pi 唤醒，不要无谓轮询 `bg_wait`。
- 所有通道结束后：交叉核对各子代理结果，去重、仲裁冲突，由**父代理**执行修复与最终验收。
- 审查类通道要求给出文件:行号证据与结论；无证据的发现不采信。
- 交付说明包含：各通道做了什么、关键结果、验证方式、残余风险；涉及文档同步时遵循全局 AGENTS.md 第 3 节。

## 红线

- 子代理默认不再派生孙代理；不得擅自下放 fanout。
- 子代理工作流启动失败属于通道基础设施故障：停下、报告确切失败与 run/worktree 状态，不得静默改用其他执行模式兜底。
- 未获用户明确授权，不执行 `git commit` / `git push`（vault 文档仓库按全局规则豁免）。
- 保持父代理的决策权与发布权；无法仲裁的分歧升级给用户，不擅自拍板。
