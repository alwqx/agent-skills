---
name: squash
description: 使用英文 commit message 和 CHANGELOG 条目，将当前功能分支相对主分支的多个提交压缩为一个提交，并在用户授权范围内更新远端分支，不创建备份分支。用户调用 /squash、$squash 或 /skill:squash、要求压缩提交或整理分支提交历史时使用；不用于合并 PR 或在主分支改写历史。
disable-model-invocation: true
---

# Squash Commits

将当前功能分支的多个提交整理成一个完整提交，保留最终代码并记录原 HEAD 的 SHA，不创建备份分支。遵循当前 agent 适用的仓库说明（AGENTS.md、CLAUDE.md 等）、提交格式及 CHANGELOG 约定。本技能不负责合并 PR。

## 语言要求

- 压缩后的 commit message 标题和正文均使用英文，保留仓库要求的类型、scope 等格式，例如 `feat(desktop): add candlestick history with volume`。
- 本次分支变更对应的 CHANGELOG 条目使用英文；已有对应中文条目时，将这些条目改写为英文而非新增重复条目。保留版本标题、编号列表等结构，不翻译无关的历史条目。
- 以上是用户明确指定的语言偏好；即使仓库旧说明或历史记录使用中文，压缩后的 commit message 与本次 CHANGELOG 条目仍使用英文。用户后续有新的明确要求时以新要求为准。
- 会话或系统提示给出 commit / PR 署名行时，按其要求原样附加在末尾；署名行不受上述语言要求影响。

## 执行前检查

1. 查看 `git status --short`、当前分支、远端和跟踪分支。工作区及暂存区必须干净，包括未跟踪文件；有未提交改动时先说明并请用户决定如何处理，不擅自 stash、丢弃或将它们混入 squash。
2. 使用 `git symbolic-ref refs/remotes/<remote>/HEAD` 确定默认分支；无法解析时检查远端配置、本地 `main` / `master`。仍不明确时询问用户。优先使用当前分支配置的远端，否则使用实际存在的 origin；多个远端无法判断时询问。
3. 若处于 detached HEAD、默认分支、`main` 或 `master`，停止并说明原因。不得在这些分支上 reset 或强制推送。
4. 获取所需远端引用，使用 `git merge-base HEAD <remote>/<base>` 找到分叉点；没有远端引用时可使用已确认的本地主分支。记录原 HEAD、分叉点、提交列表、完整差异和当前代码树。分叉点到 HEAD 没有提交或只有一个提交时说明无需压缩，不创建空提交。
5. 若功能分支已推送，记录这次获取到的远端分支 SHA。确认远端提交是本地原 HEAD 的祖先；若远端有本地未包含的提交，停止并说明，不能靠重试强制推送覆盖它们。

## 整理本地提交

1. 不创建备份分支，记录原 HEAD 的 SHA 以便恢复。先阅读待压缩的提交和最终差异，拟定能概括全部变更的英文提交说明。
2. 在确认的功能分支执行 `git reset --soft <分叉点SHA>`。必须使用 `--soft`，以保留原 HEAD 的最终内容在暂存区；不使用普通 mixed reset 或 `--hard`。
3. 复核暂存差异与原分支完整差异一致。仓库要求 CHANGELOG 时，确认其中以英文准确汇总本次变更；将本次对应中文条目改写为英文，并仅补充缺失或不准确的内容，不重复条目、不为 squash 本身增加版本。只暂存本次明确修改的文件，不使用 `git add -A` 收集其他内容。
4. 使用适当类型和英文整体摘要创建一个新提交，标题和正文均为英文，不把多次修补的对话历史拼进提交说明。
5. 对比新 HEAD 与原 HEAD 的代码树，确保除已说明的 CHANGELOG 等文档调整外没有内容变化。复用仍有效的测试结果；只有代码变化或存在具体疑虑时运行相应检查。失败时说明现状和原 HEAD 的 SHA，不继续推送错误结果。

## 同步远端

- 用户调用 `/squash`、`$squash`、`/skill:squash` 或明确要求压缩当前功能分支时，默认授权在完成上述检查后，使用带执行前记录的明确旧 SHA 的 `--force-with-lease` 更新该功能分支及其已有 PR，无需再单独询问远端历史重写授权。此偏好仅适用于当前 squash 任务；不扩展到其他分支、默认分支或合并 PR。
- 用户明确要求仅本地 squash、暂不推送或不改写远端时，遵守该限制。工具要求的权限审批仍须遵守。
- 未推送过的分支按授权执行普通 `git push -u <remote> <branch>`。
- 已推送过的分支使用执行前记录的远端 SHA 作为明确 lease：

  ```bash
  git push --force-with-lease=refs/heads/<branch>:<记录的远端SHA> <remote> HEAD:refs/heads/<branch>
  ```

  不使用 `--force`，不依赖可能被后台 fetch 更新的隐式 lease。若 lease 失败，停止并报告远端已变化；不得刷新期望 SHA 后自动重试覆盖他人的更新。
- 不创建或合并 PR，已有 PR 会随分支推送自动更新。操作结束简要报告压缩前后的提交数量、新提交 SHA、原 HEAD 的 SHA、是否已推送以及未完成事项。

来源：个人 opensec 项目的 squash 技能，整合 Claude 与 Codex 版本；保留 soft reset、明确 lease 和不创建备份分支的约定。
