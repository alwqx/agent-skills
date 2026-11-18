# Personal Agent Skills

集中维护个人技能，供 Claude Code、Codex 和 pi 等 agent 复用。`skills/` 是本仓库唯一的技能源目录；每个技能只保留一个版本。

## 技能

| 技能 | 用途 | 依赖 |
| --- | --- | --- |
| [pr](skills/pr/SKILL.md) | 英文提交、更新 CHANGELOG、推送并创建 PR | Git、GitHub CLI |
| [squash](skills/squash/SKILL.md) | 压缩功能分支提交，按授权更新远端 | Git |
| [merge](skills/merge/SKILL.md) | 检查并合并 PR，同步 main、清理已合并分支 | Git、GitHub CLI |
| [fix-review](skills/fix-review/SKILL.md) | 逐项修复 PR 行级评审意见，验证、提交、推送并解决线程 | Git、GitHub CLI、项目构建工具 |

## 使用与维护

在每台电脑克隆这个仓库，修改 `skills/<name>/` 后通过 Git 提交、推送；其他电脑使用 `git pull --ff-only` 获取更新。

本地 agent 的技能入口可以逐个链接到仓库中的技能目录：

- Claude Code：`~/.claude/skills/<name>`。
- Codex 和支持共享技能目录的 pi：`~/.agents/skills/<name>`。

例如，对尚不存在的入口安装 `pr`（在仓库根目录运行）：

```sh
mkdir -p ~/.agents/skills ~/.claude/skills
ln -s "$PWD/skills/pr" ~/.agents/skills/pr
ln -s "$PWD/skills/pr" ~/.claude/skills/pr
```

先检查目标是否已有文件、目录或链接；已有版本需要比较和迁移，不能直接覆盖。尤其不要同时在 `~/.codex/skills` 和 `~/.agents/skills` 安装同名副本。链接创建后，按 agent 的加载机制刷新或重新开启会话。

调用示例：Claude Code 使用 `/pr`，Codex 使用 `$pr`，pi 使用 `/skill:pr`；也可以按各技能的触发说明描述任务。`merge` 和 `squash` 保留 Claude 的显式调用设置。不同 agent 对 frontmatter 的支持并不完全相同，调用策略应在目标环境核实。`agents/openai.yaml` 保留 Codex 的显示信息，不参与通用工作流程。

## 本次整理记录

2026-11-18 从 `~/.claude/skills` 和 `~/.codex/skills` 整理，原目录未修改。

| 技能 | 源文件与去重方式 |
| --- | --- |
| pr | 以 Claude 版本为基础，合并 Codex 的 `agents/openai.yaml`；保留署名要求和来源说明，统一调用方式与仓库说明文件表述。此技能由官方 commit-commands 流程个人定制而来，并非原样导入官方插件。 |
| squash | 以较新的 Codex 版本为基础，保留不创建备份分支和明确 SHA lease 的规则；合入 Claude 的署名要求与 `disable-model-invocation` 字段，保留 Codex UI 元数据。 |
| merge | 来自 Claude；保留现有合并和清理规则，将 `$ARGUMENTS` 替换为从用户请求读取 PR 编号，以便跨 agent 使用。 |
| fix-review | 来自 Codex；保留修复和验证流程，补充跨 agent 的调用与仓库说明表述。 |

原文件路径与 SHA-256 记录在 [migration-sources.json](migration-sources.json)，用于追溯导入来源；此文件不参与技能发现。

本次不纳入：

- `xiaoxiaodong-vps`：按用户要求暂不处理。
- Codex `.system`、官方分发目录、插件缓存，以及 Claude 的平台同步技能。
- Claude `.trash` 中已删除的内容。
- Codex 的 `sec` 空目录：没有 `SKILL.md`。

此次只整理仓库内容，尚未替换本机入口、创建软链接或提交推送。
