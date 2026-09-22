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
- worktree 生命周期与清理能力：`docs/workflows.md`、`docs/tool-reference.md`

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

并行实现必须让代码真正回到**启动子代理前父代理所在的原分支与原工作区**，并清理本次
隔离执行留下的临时现场；回交、验证、清理全部完成后，才能报告任务完成。

0. **记录原现场**：派发前记录仓库根目录 `originalCwd`、完整分支引用 `originalBranch`
   （用 `git symbolic-ref --quiet HEAD` 获取，不能假定为 main/master）、`originalHead`、
   工作区/暂存区状态和 `git worktree list --porcelain`。detached HEAD 时先确认目标分支。
   使用 managed worktree 前检查源工作区干净；有既有改动时保留并报告阻塞，不擅自
   stash、commit 或 reset。每条 lane 分配后记录实际 worktree 路径、临时分支和本次
   新建的临时产物清单，清理范围以运行时 handoff 与该清单交叉核实。
1. **可安全并行的实现**：隔离可用 `worktree: true`；并行可写 worker 必须使用独立
   worktree，每个 worker 声明唯一的 `claimed files or contract`。任务包包含原分支、
   原工作区和基线，明确子代理只写自己的工作区、不切换原分支、不自行合并或清理现场，
   默认禁止 commit/push。返回 changed files、验证结果、未决事项；父代理从运行时
   结果取得 `artifactPaths` 中的 handoff/patch 引用，不要求子代理猜测运行结束后才生成的路径。
2. **父代理合并回原分支**：所有实现 lane 结束后，父代理在 `originalCwd` 核实当前分支
   仍为 `originalBranch`，并比对 HEAD 与原有改动；发生意外切换或外部改动时先核对，
   不向当时恰好所在的其他分支应用结果，不自动切换覆盖用户现场。逐一检查 handoff、
   patch 的范围和完整性（包括新增、删除、重命名、二进制与未跟踪源码），按依赖顺序
   应用已接受的变更，每个 patch 先 `git apply --check` 再应用，解决冲突后检查最终 diff。
   默认通过 patch 把代码合入原分支的工作区，保留为未提交变更；只有另获提交授权时才
   使用会创建提交的 Git merge/cherry-pick。必须把原分支作为最终交付位置，不能只留下
   子分支、worktree 或 patch，也不能根据子代理口头描述重写成果。
3. **原分支验证**：父代理在 `originalCwd` 对最终合并结果运行受影响范围的 LSP 诊断与
   构建/测试，并检查 diff 和原分支身份。子代理 worktree 中通过的命令只是 lane 证据，
   不能替代最终验证；不适用或无法运行的检查必须说明原因，不能伪称通过。
4. **清理现场**：原分支变更完整且验证通过后，按下一节删除本次创建的隔离 worktree、
   临时分支和临时文件/目录，确认无遗留；清理是完成条件，不能仅在交付中建议用户自行做。
5. **冲突与失败**：lane 失败、合入失败、验证失败或清理失败时，保留尚需恢复的成果和
   handoff，标记 blocked，列出保留路径、原因和下一步；修复后继续合入、验证和清理。
   不得为清理而丢弃未整合成果，不得静默切换执行模式，不得报告整体完成。
6. **不能安全分片时**：使用“并行只读调研/审查 → 一个 worker 串行实现 → 父代理验证”，
   这仍然是有效编排；使用隔离目录时同样必须回到原分支并清理。

父代理负责 patch 整合、冲突仲裁、最终验证和清理；这些步骤属于编排收尾。

## worktree 与临时产物清理

- **先核验所有权与成果**：确认 workflow 和全部子代理已终止、无进程/后续步骤占用目标；
  handoff 对应当前 run，路径和临时分支确为本次新建。把最终变更、验证结果和清理清单
  摘要保存在隔离目录外。核对每条 lane 的成果均已进入原分支，不能仅凭退出码或报告就删除。
- **适配运行时自动清理**：pi-subagents 可能在捕获 patch 后已移除 worktree 和临时分支；
  此时依赖保留的 patch 完成合入，并对照 handoff、Git worktree 列表和文件系统确认已移除，
  不重复删除。运行时“cleanup complete”不代表代码已合入原分支。
- **处理残留 worktree**：对尚存在且核验安全的目录，优先采用当前版本支持的清理机制；
  注意 `worktree.cleanup` 当前仅支持 `mode: "plan"`，计划成功不等于删除完成，不编造
  `mode: "apply"`。在已授权且完成所有权/成果核验后，可用
  `git -C "$originalCwd" worktree remove "$laneWorktree"` 移除干净的本次 worktree，
  再用 `git -C "$originalCwd" branch -d "$laneBranch"` 删除本次临时分支。命令拒绝时
  重新检查未提交文件、忽略文件及未合并提交；patch 合入不一定建立 Git 祖先关系，
  不能仅为让 `-d` 成功而制造提交。
  需要丢弃残留内容时按当前版本的 `worktree.discard` 与权限机制处理，不擅用强制删除绕过检查。
- **删除本次临时文件**：对照新建清单清理 worktree 目录中的构建产物、缓存、临时报告和
  本次创建的外部临时目录/文件。worktree 根目录必须由 Git/受支持清理机制移除，不能
  直接递归删除造成残留注册。只删除核实归属于本次运行的精确路径；符号链接只移除链接，
  不跟随删除共享依赖或缓存。原工作区、原分支、既有文件、其他任务的 worktree 和正式
  交付源码均不在删除范围；不使用全仓 `git clean`、宽泛通配符或整个共享 artifacts 目录清扫。
- **保留必要记录并复核**：恢复不再需要的本次 patch/临时报告可在最终验证与清理摘要保存后
  删除；保留 runtime 管理的 handoff/receipt/会话记录，遵循其保留机制，不破坏恢复索引。
  再次检查 `git worktree list --porcelain`、临时分支列表及精确路径是否存在，并在
  `originalCwd` 核对原分支与最终 diff。存在无法处理的残留时标记清理 blocked，列出具体路径。

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

分阶段（并行侦察 → 并行实现 → 父代理合入原分支、验证与清理）：

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
    task: "...原分支与原工作区、基线、claimed files or contract；禁止 commit/push 与自行合并清理；返回 changed files、验证结果、临时产物清单...",
    worktree: true,
    output: "handoff/impl-a.md",
    outputMode: "file-only",
  },
  {
    key: "impl-b",
    agent: "worker",
    phase: "Implementation",
    label: "实现 B",
    task: "...原分支与原工作区、基线、claimed files or contract；禁止 commit/push 与自行合并清理；返回 changed files、验证结果、临时产物清单...",
    worktree: true,
    output: "handoff/impl-b.md",
    outputMode: "file-only",
  },
]);
return impl.map(({ key, runId, outputReference, artifactPaths }) => ({
  key, runId, outputReference, artifactPaths,
}));
```

脚本返回 handoff 引用后，父代理在脚本外依次核实原分支、顺序应用 patch、解决冲突、
运行最终 LSP/构建/测试、清理本次 worktree 与临时产物、复核原分支和残留路径；
只有变更已合入原分支、验证与清理均完成，才能向用户报告实现完成。

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
- 全部通道结束后：父代理交叉核对、仲裁冲突，依据 runtime handoff/patch 将已接受
  变更整合回启动前记录的原分支与原工作区，再执行最终 LSP、构建/测试及 diff 检查，
  最后清理本次 worktree、临时分支和临时产物并复核。小规模收尾修改允许父代理直接做。
- 审查类通道必须给出文件:行号证据；无证据的发现不采信。
- 交付说明必须包含：通道计划与实际执行对照、各通道结果、原分支与合入状态、
  最终验证结果、清理清单和残留路径/原因；涉及文档同步时遵循全局 AGENTS.md 第 3 节。

## 红线

- 子代理默认不再派生孙代理；不得擅自下放 fanout。
- 子代理工作流启动失败属于通道基础设施故障：停下、报告确切失败与
  run/worktree 状态，不得静默改用其他执行模式。
- 可写 lane 的 handoff、patch 或 worktree 状态无法确认时，不得把文字报告当作代码已
  回交；保留现场并报告阻塞原因。
- 未获用户明确授权，不执行 `git commit` / `git push`（vault 文档仓库按
  全局规则豁免）。
- 保持父代理的决策权与发布权；无法仲裁的分歧升级给用户，不擅自拍板。
