---
name: merge
description: 检查指定 GitHub PR 能否合并；可以合并时合并 PR、切换到 main、拉取最新代码并删除已合并的本地和远端分支，不能合并时输出原因。仅在用户调用 /merge、$merge 或 /skill:merge 并指定 PR 编号时使用。
argument-hint: <PR编号>
disable-model-invocation: true
---

# 合并 PR

PR 编号：从用户调用参数或当前请求中读取。支持 `/merge <PR编号>`、`$merge <PR编号>` 和 `/skill:merge <PR编号>`；不要依赖特定 agent 的参数替换变量。

检查通过后，依次执行：合并 PR → `git checkout main` → `git pull` → `git branch -d <已合并分支>` → 删除远端分支。任何检查不通过时不合并，只输出原因。

## 1. 检查 PR

1. 编号必须是单个正整数。缺失或无效时，提示用法 `/merge <PR编号>` 并结束。
2. 读取 PR 状态：

   ```bash
   gh pr view <PR编号> --json number,title,url,state,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,baseRefName,headRefName,headRefOid,isCrossRepository
   ```

3. `mergeable` 或 `mergeStateStatus` 为 `UNKNOWN` 时，表示 GitHub 仍在计算。每隔约 10 秒重新读取，最多 3 次；仍为 `UNKNOWN` 时按不能合并处理。
4. 以下条件全部满足才可以合并：

   | 字段 | 要求 |
   | --- | --- |
   | `state` | `OPEN` |
   | `isDraft` | `false` |
   | `baseRefName` | `main` |
   | `mergeable` | `MERGEABLE` |
   | `mergeStateStatus` | `CLEAN` 或 `HAS_HOOKS` |
   | `reviewDecision` | 不是 `CHANGES_REQUESTED` 或 `REVIEW_REQUIRED` |
   | `statusCheckRollup` | 没有失败或未完成的检查 |

5. 检查本地仓库：工作区和暂存区必须干净（`git status --porcelain` 为空，包括未跟踪文件），否则切换分支可能失败或带上无关改动。

## 2. 不能合并时

不执行任何合并或分支操作，输出：

- PR 编号、标题和链接。
- 每个未满足的条件及实际值，例如 `mergeStateStatus=BLOCKED`。
- 失败或未完成的检查名称。
- 简短说明需要做什么，例如解决冲突、等待 CI、完成 review 或清理工作区。

## 3. 合并并同步本地

按顺序执行，某一步失败就停止，并说明已完成的步骤和失败原因：

1. 合并 PR。仓库现有历史使用 merge commit，所以使用 `--merge`；`--match-head-commit` 确保合并的就是刚检查过的提交：

   ```bash
   gh pr merge <PR编号> --merge --match-head-commit <headRefOid>
   ```

2. `git checkout main`
3. `git pull --ff-only`
4. 确认合并结果：`gh pr view <PR编号> --json state,mergedAt,mergeCommit` 中 `state` 为 `MERGED`。
5. 删除本地分支：`git branch -d <headRefName>`。

   - `isCrossRepository=true`，或本地没有这个分支时，说明原因并跳过。
   - `-d` 因分支未完全合并而失败时，保留分支并报告，继续下一步。

6. 删除远端分支：先用 `git ls-remote --heads origin <headRefName>` 读取远端分支的 SHA，再删除：

   ```bash
   git push origin --delete <headRefName>
   ```

   - `isCrossRepository=true` 时，分支在别人的 fork 上，说明原因并跳过。
   - `headRefName` 是 `main`、`master` 或仓库默认分支时，不删除并报告。
   - 远端分支已不存在时（例如仓库开启了合并后自动删除），说明后跳过。
   - 远端分支的 SHA 与 `headRefOid` 不一致时，说明合并后又有新提交，保留分支并报告。
   - 还有其他打开的 PR 以该分支为 base 时（`gh pr list --base <headRefName> --state open`），保留分支并报告，避免这些 PR 被关闭。

## 执行边界

- 不使用 `--admin`、`--auto`，不绕过分支保护。不能合并就报告，不改成等待自动合并。
- 不使用 `git branch -D`，不强制推送，不改写历史。
- 只删除本次合并 PR 的头分支（本地和远端）；不带 `--delete-branch`，合并确认成功后才删除。不删除备份分支或其他分支。
- 不擅自 stash、丢弃或提交本地改动。
- 仓库禁用 merge commit 时报告即可，不自行改用 squash 或 rebase。

## 结束时报告

简要说明：PR 链接、合并提交的短 SHA、当前分支和最新提交、已删除的本地和远端分支，以及仍未处理的事项（如因上述原因保留的分支）。
