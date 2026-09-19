# pi-subagents-skill

给 [pi](https://github.com/badlogic/pi-mono) 的 `pi-subagents` 插件配套的 swarm 技能包：
输入 `/swarm <任务>`，父代理自动把任务分解为有边界的多条子代理通道，
并行/分阶段派发、回收并汇总交付。

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
# 方式一：git 远程安装（推荐，远程仓库创建后）
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
父代理不独自执行任务本体。仅当任务原子不可分时才缩减为单子代理，且需在交付说明中写明理由。
全部通道完成后由父代理交叉核对、仲裁冲突、统一验收交付。

也可以直接 `/skill:swarm <任务>` 触发同一 skill。

## 卸载

```bash
pi remove git:github.com/baozi-2019/pi-subagents-skill   # 或本地路径安装时的对应来源
# 方式三安装则删除上述两个软链
```
