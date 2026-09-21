---
name: swarm
description: |
  多子代理群（swarm）编排：把一个复杂任务强制分解为有边界的多条子代理通道，
  通过 pi-subagents 的 subagent 工具 workflowScript 并行派发、回收并汇总结果。
  当用户输入 /swarm <任务>、或明确要求"用多个子代理/并行派发/swarm 方式"完成任务时使用。
---

# Swarm：多子代理并行编排

用户输入 `/swarm <任务>` 即构成**明确的委派授权**，并且要求的是**多代理并行**
而不是父代理单干。本 skill 是父代理（你）的编排手册：你是编排者，负责
分解、派发、仲裁、验收与交付；子代理执行，父代理不直接做执行任务本身。

开始编排前，若对 `subagent` 工具字段或 `runs.run` / `runs.all` / `runs.lanes`
用法不确定，先读取 pi-subagents skill 的参考文档（路径相对于
`~/.pi/agent/npm/node_modules/pi-subagents/`）：

- 角色与提示词：`skills/pi-subagents/references/prompting-and-roles.md`
- 执行控制（async、workflowScript 等）：`skills/pi-subagents/references/execution-controls.md`

也可调用 `subagent({action:"guide",topic:"workflows"})` 获取权威用法。

## 硬性要求：先拆分、再并行

1. **父代理禁止独自执行任务本体**。收到 `/swarm` 任务后第一步必须是通道分解，
   不是自己动手。
2. **默认并行扇出**：将任务分解为 2~4 条相互独立的通道，用**一个**
   `subagent` 调用的 `workflowScript` + `runs.all([...])` 并行派发，默认 async。
3. 仅当任务**原子不可分**（如改一行配置、回答一个事实性问题）时才允许
   缩减为单子代理，且必须在交付说明中写明"不可分"的理由；禁止以"任务小"
   为由静默跳过 swarm 编排——任务小就不该用 `/swarm`，应提示用户直接下达。
4. 派发前先输出**通道计划**再立即执行（无需等用户确认；仅当分解存在
   实质性歧义时才先 `ask_user_question`）：

   | key | label | agent | 职责 | 读写边界 |
   | --- | --- | --- | --- | --- |
   | scout-auth | 调研鉴权模块 | scout | 梳理鉴权链路并输出问题清单 | 只读 |
   | scout-perm | 调研权限模型 | scout | 梳理权限模型与数据契约 | 只读 |

## 分解原则

- **按接缝拆分**：不同模块/目录/调研角度/验收维度各成一条通道，每条有独立的
  证据来源或修改范围；禁止只换序号的克隆提示词。
- **先判定能否并行实现**：只有当通道拥有互不重叠的文件集合或稳定的契约边界，且
  一条通道的中间结果不会改变另一条通道的实现前提时，才并行派发可写 worker。
  共享接口、同一文件、同一迁移序列或存在先后依赖时，先并行只读调研，再由父代理
  重写任务包并串行实现；不要用 worktree 掩盖语义冲突。
- 常见可并行维度：不同子系统、侦察 vs 实现 vs 审查、不同文件集合、
  不同假设方案的对照验证。
- 2~4 条通道是常见上限；超出时合并相近通道，不制造并行。
- 有依赖关系时按"阶段 × 并行"组织：每个阶段内部 `runs.all` 并行，
  阶段之间顺序 await（例如：并行侦察 → 父代理汇总重写任务包 → 并行实现 →
  fresh-context 并行审查）。

## 并行实现与回交协议

并行实现必须让代码真正回到父代理当前工作区，子代理的文字报告不能视为交付完成。
按以下协议选择实现形态：

1. **可安全并行的实现**：每个 worker 声明唯一的 `claimed files or contract`，使用
   `worktree: true` 获得独立 worktree；任务包明确禁止 commit/push，并要求返回 changed
   files、验证命令/结果、未决事项以及运行时提供的 `handoffPath`、`artifactPaths` 或
   patch 引用。不要让多个 worker 共用一个可写 cwd。
2. **父代理整合**：所有实现 lane 结束后，父代理按 lane 状态和 handoff 清单逐一检查
   diff，再把已接受的 patch/变更按顺序应用到自己的当前工作区；每次应用后检查文件归属
   和冲突。父代理只能整合已完成且证据完整的 lane，不能根据子代理口头描述重写结果。
3. **整合后验证**：代码进入父代理当前工作区后，父代理必须针对最终合并结果运行受影响
   范围的 LSP 诊断与构建/测试，并检查最终 diff。子代理在独立 worktree 中通过的命令
   只能作为 lane 证据，不能替代父代理对最终工作区的验证。
4. **冲突与失败**：patch 无法应用、lane 失败、worktree 状态不明或验证失败时，保留
   该 lane 的 worktree 和 handoff，标记为 blocked，停止宣称整体完成；由父代理在清晰的
   同协议重试、串行修复或向用户升级之间作决定。不得静默切换成另一种执行模式。
5. **不能安全分片时**：使用“并行只读调研/审查 → 一个 worker 串行实现 → 父代理验证”，
   这仍然是有效编排；不要为了满足并行字面要求而制造冲突写者。

父代理禁止独自执行任务本体的要求，不阻止父代理做上述 patch 整合、冲突仲裁和最终
验证；这些步骤是把子代理结果交回主工作区并完成验收的编排收尾。

## 派发模板

单次并行扇出（workflowScript 是普通 JS 语句体：顶层 await、显式 return，
不用嵌套 async 辅助函数）：

```js
const results = await runs.all([
  {
    key: "lane-a",
    agent: "scout",
    label: "调研 X",
    task: "目标：...\ncwd：...\n权限：只读\n背景与契约：...\n验收标准：...\n验证方式：...\n输出格式：...\n停止条件：...",
  },
  { key: "lane-b", agent: "scout", label: "调研 Y", task: "..." },
]);
return results;
```

分阶段（并行侦察 → 并行实现 → 父代理整合与验证）：

```js
const scout = await runs.all([
  { key: "scout-a", agent: "scout", phase: "Recon", label: "侦察 A", task: "..." },
  { key: "scout-b", agent: "scout", phase: "Recon", label: "侦察 B", task: "..." },
]);
// 父代理基于 scout 结果重写实现任务包，并为每个 worker 分配不重叠的文件/契约边界。
const impl = await runs.all([
  {
    key: "impl-a",
    agent: "worker",
    phase: "Implementation",
    label: "实现 A",
    task: "...明确 claimed files or contract；禁止 commit/push；返回 changed files、验证结果和 handoff 引用...",
    worktree: true,
    output: "handoff/impl-a.md",
    outputMode: "file-only",
  },
  {
    key: "impl-b",
    agent: "worker",
    phase: "Implementation",
    label: "实现 B",
    task: "...明确 claimed files or contract；禁止 commit/push；返回 changed files、验证结果和 handoff 引用...",
    worktree: true,
    output: "handoff/impl-b.md",
    outputMode: "file-only",
  },
]);
return impl.map(({ key, runId, outputReference, artifactPaths }) => ({
  key, runId, outputReference, artifactPaths,
}));
```

脚本返回 handoff 引用后，父代理在脚本外按“检查 lane → 顺序应用 patch → 解决冲突 →
运行最终 LSP/构建/测试”的顺序收尾；只有最终代码已进入父代理当前工作区并通过验证，才能向
用户报告实现完成。

约束：

- 每个 `runs.run` / `runs.all` 条目必须给出稳定 `key` 与短动词短语 `label`。
- 每个子代理任务包必须**冷启动完备**（8 项）：目标 / 仓库与 cwd、ref /
  权限边界（只读或可写范围）/ 相关文件与契约 / 验收标准 / 验证方式 /
  输出格式 / 停止与求助条件。子代理看不到父会话历史，任务包即其全部上下文。
- **一个写者一个目录**：多条可写通道不得共用同一 cwd/worktree，用
  `worktree: true` 隔离或改为串行；只读通道无此限制。
- 工作者/侦察走快而够用的模型档，正式审查走强模型档；父代理保持默认强模型。

## 回收与汇总

- 异步通道有原生完成通知；派发后让出控制权等 Pi 唤醒，不做无谓轮询。
- 全部通道结束后：交叉核对结果、去重、仲裁冲突；对可写 lane，父代理必须先依据
  `handoffPath` / `artifactPaths` 检查并把已接受的 patch 或变更整合进当前工作区，随后
  执行最终 LSP 诊断、受影响范围的构建/测试和 diff 检查。修复与最终验收由**父代理**
  执行；小规模收尾修改允许父代理直接做。
- 审查类通道必须给出文件:行号证据；无证据的发现不采信。
- 交付说明必须包含：通道计划与实际执行对照、各通道关键结果、代码回交/整合状态、
  最终验证方式、残余风险；涉及文档同步时遵循全局 AGENTS.md 第 3 节。

## 红线

- 子代理默认不再派生孙代理；不得擅自下放 fanout。
- 子代理工作流启动失败属于通道基础设施故障：停下、报告确切失败与
  run/worktree 状态，不得静默改用其他执行模式。
- 可写 lane 的 handoff、patch 或 worktree 状态无法确认时，不得把文字报告当作代码已
  回交；保留现场并报告阻塞原因。
- 未获用户明确授权，不执行 `git commit` / `git push`（vault 文档仓库按
  全局规则豁免）。
- 保持父代理的决策权与发布权；无法仲裁的分歧升级给用户，不擅自拍板。
