# pi-subagents-skill

给 [pi](https://github.com/badlogic/pi-mono) 的 `pi-subagents` 插件配套的 swarm 技能包：
输入 `/swarm <任务>`，父代理自动把任务分解为有边界的多条子代理通道，
并行/分阶段派发、回收并汇总交付。

仓库地址：`git@github.com:baozi-2019/pi-subagents-skill.git`

## 组成

```text
├── skills/swarm/SKILL.md   # swarm 编排手册（父代理的分解/派发/汇总规则）
├── prompts/swarm.md        # /swarm 提示词模板（命令入口，触发加载 swarm skill）
└── package.json            # pi 包清单（pi.skills / pi.prompts）
```

- `swarm` skill：编排 playbook——定界、按接缝分解通道、async 派发
  冷启动完备的任务包、回收汇总、父代理验收。也会作为 `/skill:swarm` 注册。
- `/swarm` prompt template：展开为"加载 swarm skill 并编排以下任务"的指令，
  提供 `/swarm <任务>` 的原生用法。

## 安装

前置条件：已安装 `pi-subagents` 包（提供 `subagent` 工具与各内置 agent）。

```bash
# 方式一：git 远程安装（推荐）
pi install git:github.com/baozi-2019/pi-subagents-skill

# 方式二：本地路径安装（写入 ~/.pi/agent/settings.json）
pi install /Users/baozi/study/pi-extensions/pi-subagents-skill

# 方式三：手动软链
ln -s "$PWD/skills/swarm" ~/.pi/agent/skills/swarm
ln -s "$PWD/prompts/swarm.md" ~/.pi/agent/prompts/swarm.md
```

## 用法

```text
/swarm 调研本仓库的鉴权链路并输出问题清单
/swarm 把 apps/web 和 apps/api 的 lint 错误分别修掉，再互相交叉审查
```

输入 `/swarm` 即构成明确的委派授权：父代理**必须先输出通道计划**，把任务拆成
2~4 条相互独立的子代理通道，再用一个 `workflowScript` + `runs.all` **并行派发**（默认异步）；
可写通道必须按不重叠的文件/契约分片，多个并行 worker 使用独立 worktree。父代理启动前
记录原工作区、原分支与 HEAD，跟踪本次创建的 worktree、临时分支及临时产物。子代理完成后，
父代理通过 runtime handoff/patch 取得实际变更，按顺序合入**启动前所在的原分支**，
在原工作区执行最终 LSP、构建/测试与 diff 检查；默认以未提交变更交回，commit/push 仍须授权。
验证通过、子代理已退出且成果保留完整后，必须清理本次 worktree、临时分支和临时文件/目录，
核实 Git worktree 注册及路径均无残留，保护原工作区、正式源码与必要运行记录。
**合入、验证、清理全部完成才算任务完成**；任一步失败，保留恢复材料并报告具体阻塞与路径。
父代理负责冲突仲裁、代码整合、最终验证和清理交付。仅当任务原子不可分时才缩减为单子代理，
并在交付说明中写明理由。

也可以直接 `/skill:swarm <任务>` 触发同一 skill。

## 更新

git 方式安装的包（未固定 `@ref`）用以下命令拉取远程最新提交：

```bash
pi update git:github.com/baozi-2019/pi-subagents-skill   # 只更新本包
pi update --extensions                                   # 更新全部已安装的包
```

更新后需新开 pi 会话才会加载新版 skill。
注意：本地路径与 git 两种方式不要同时安装，同名资源会冲突且只生效先发现的一份。

## 卸载

```bash
pi remove git:github.com/baozi-2019/pi-subagents-skill   # 或本地路径安装时的对应来源
# 方式三安装则删除上述两个软链
```
