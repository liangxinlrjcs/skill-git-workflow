---
name: git-workflow
description: >
  General git workflow skill for embedded/application development projects.
  Use when the user mentions git commit, git push, git pull, branch creation, feature branch,
   commit message, code review, merge, rebase, cherry-pick, stash, conflict resolution, git tag,
   release branch, secret leak, credential cleanup, git log, git diff, git status, or any version control operation. Also trigger when the user asks
  about when to commit, how to write commit messages, branch naming rules, whether to push now,
  how to create a feature branch, how to submit a patch, or how to handle git merge conflicts.
  Special rules apply: never auto-commit or auto-push; only commit after build succeeds AND
  user confirms device/integration test passes; always fetch before commit or push to check
  if remote is ahead; never force push to main; handle stash+rebase+conflict collaboratively
  with user before merging remote changes.
argument-hint: "[operation: commit|push|pull|branch|merge|log] [--dry-run]"
---

# Git Workflow — 通用开发工作流

本 Skill 管理嵌入式/应用开发项目的 Git 工作流，核心原则：**先编译、先测试、再提交，永远不自主 push**。

> 注：编译命令、设备部署命令按实际项目而定（见「编译验证」章节），需询问用户确认当前项目类型。

---

## ⚠️ 强制规则（违反即终止操作，必须先询问用户）

| 规则 | 说明 |
|------|------|
| **禁止自主 commit** | 代码修改完成后不得自动 commit，必须用户明确确认测试通过后才能 commit |
| **禁止自主 push** | 即使 commit 完成，push 也需要用户单独明确批准 |
| **禁止 force push** | 禁止 `git push --force` 或 `git push -f`，尤其是 main/master 分支；任何 history rewrite 都需单独确认 |
| **禁止 git clean -xdf** | Yocto build 目录下禁止强制清理，会破坏 sysroot |
| **禁止 bitbake cleansstate/clean** | 见 AGENTS.md，会导致 mergerfs 权限错误损坏 sysroot |
| **禁止直接 push main** | 所有修改必须在 feature 分支完成，严禁直接推送 main/master |
| **禁止擅自删除/覆盖远端 tag/branch** | release/tag 修复属于高风险操作，必须二次确认；不要建议 force-update tag，优先创建修正 tag |
| **敏感信息先止血** | 一旦误提交 token/key/password，先停止传播并提醒轮换密钥，再讨论 Git 历史清理 |

---

## 工作流概览

```
1. 创建 feature 分支 (feature/MMDD-HHMM)
   ↓
2. 开发 + 修复 Bug
   ↓
3. Docker 编译验证 → MUST 通过
   ↓
4. adb push 到设备 + 用户测试 → MUST 用户确认通过
   ↓
5. [等待用户明确说"可以提交"] → git add + git commit
   ↓
6. [等待用户明确说"可以推送"] → git push origin feature/MMDD-HHMM
```

---

## 分支命名规则

| 类型 | 格式 | 示例 |
|------|------|------|
| 功能开发 | `feature/MMDD-HHMM` | `feature/0522-1430` |
| Bug 修复 | `feature/MMDD-HHMM` | `feature/0523-0930` |
| 热修复 | `hotfix/MMDD-HHMM` | `hotfix/0522-2100` |

```bash
# 创建并切换 feature 分支
git checkout -b feature/$(date +%m%d-%H%M)

# 查看当前分支
git branch -v

# 示例：基于 main 创建分支
git checkout main && git pull origin main
git checkout -b feature/0522-1430
```

---

## Commit Message 格式

项目固定格式（必须严格遵守）：

```
JAX Embedded Camera: <一句话描述，动词开头，<72字符>

- 变更点1：说明做了什么，为什么这样做
- 变更点2：说明做了什么，为什么这样做
- 变更点3：（如适用）

Test-Result:
1. <测试项目>: <结果描述>
2. <测试项目>: <结果描述>

IssueID: To be determined
Change-Id: <由 git hook 自动生成，或留空>
```

### 真实示例

```
JAX Embedded Camera: Fix IMU boot race with async SEESalt init

Make init() non-blocking by spawning background thread, recreate
see_salt instance on each retry to refresh sensor list cache

Test-Result:
Device reboot verified, IMU sensors found on 2nd attempt (~3s),
service startup not blocked, 500Hz streaming OK

IssueID: To be determined
Change-Id: Ic98c3dd7819ba0c23a557fd1efba9c757b1b0aa7
```

### 类型前缀参考（subject 第一个动词）

| 场景 | 推荐动词 |
|------|----------|
| 新功能 | Add, Implement, Support |
| Bug 修复 | Fix, Resolve, Correct |
| 重构 | Refactor, Simplify, Extract |
| 性能 | Optimize, Reduce, Improve |
| 配置/文档 | Update, Revise |
| 删除/清理 | Remove, Clean up |

---

### Commit 安全边界

| 场景 | 推荐处理 | 规则 |
|---|---|---|
| 未 push，漏提交文件 | 可在用户确认后 `git commit --amend` | amend 前先展示 diff/status |
| 已 push，需要修正 | 新增修复 commit 或 `git revert` | 不重写远端历史 |
| CI 失败 | 小修可 amend（未 push），已 push 则新 commit | 保持远端历史稳定 |
| 大改动混在一个 commit | 未 push 时可 interactive rebase 拆分 | 操作前需用户确认 |
| 提交了二进制/大文件 | 暂停并让用户确认处理策略 | 不擅自 reset/clean |


## 编译验证（commit 的前提条件）

**仅当编译通过后才能请用户确认是否 commit。** 编译方式取决于具体项目，使用前先确认。

> 详细编译命令和设备部署流程见 [references/build-and-deploy.md](references/build-and-deploy.md)

核心原则：
- 若不确定编译方式，先问用户
- 禁止 `bitbake -c cleansstate/clean`
- 编译通过 ≠ 功能正常，还需设备测试

---

## Commit 操作（仅用户确认后执行）

> ⚠️ Agent 永远不可以自主 commit。必须用户明确说"可以提交"。

### 流程

1. `git fetch origin` — 确认远端无新提交
2. 若远端有新提交 → 先执行 [Pull+Rebase 流程](references/pull-rebase-workflow.md)
3. `git add -p` — 交互式 stage（推荐，避免误提交）
4. `git commit` — 按格式填写 message
5. 等待用户下一步指令

```bash
git fetch origin
git log HEAD..origin/main --oneline  # 有输出=远端领先，需先 rebase
git add -p                           # 交互式逐块 stage
git commit                           # 打开编辑器写 message
```

---

## Push 操作（仅用户明确批准后执行）

> ⚠️ 即使 commit 完成，push 也需用户单独明确批准。

```bash
git fetch origin
git log HEAD..origin/$(git branch --show-current) --oneline  # 远端是否领先
git push origin $(git branch --show-current)                  # push 当前分支

# 首次 push（设置 upstream）
git push -u origin $(git branch --show-current)
```

---

## 常用 Git 操作速查

```bash
git log --oneline -20              # 最近 20 条
git show <commit>                  # 查看某提交
git log --follow -- <file>         # 文件变更历史
git stash push -m "WIP: xxx"      # 暂存当前工作
git stash pop                      # 恢复
git diff HEAD --stat               # 当前改动概览
git restore --staged <file>        # 取消 stage
git restore <file>                 # ⚠️ 丢弃工作区改动（不可恢复）
```

---

## Cherry-pick（必须走 PR，禁止直接 push main）

核心规则：
1. Cherry-pick 后**必须重新编译验证**
2. **禁止直接 push main**，push 到 feature 分支后通过 PR 合并
3. 使用 `-x` 记录来源

```bash
git cherry-pick -x <commit-hash>
# 冲突时：解决 → git add → git cherry-pick --continue
```

> 详细操作见 [references/cherry-pick-guide.md](references/cherry-pick-guide.md)

---

## 冲突解决

核心原则：**每个冲突必须与用户协同解决，不自动选择一方。**

```bash
git status                    # 查看冲突文件
# 编辑冲突文件后：
git add <resolved-file>
git rebase --continue         # 或 git merge --continue
```

> 详细流程见 [references/conflict-resolution.md](references/conflict-resolution.md)

---

## Pull + Rebase 工作流

> 场景：远端有新提交，本地有 WIP 修改需要同步。

概要步骤：`stash → fetch → rebase → 解决冲突 → pop stash → 重新编译`

> 完整 6 步流程见 [references/pull-rebase-workflow.md](references/pull-rebase-workflow.md)

---

## 分支生命周期

核心规则：短命 feature 分支优先，长期共享分支谨慎 rebase。

1. 创建前：从 `main` 或远端最新基线创建，分支名使用 `feature/MMDD-HHMM` 或 `hotfix/MMDD-HHMM`
2. 开发中：频繁 `git fetch`，避免长时间落后
3. 合并前：编译、设备测试、code review 均通过
4. 合并后：确认远端已合并再删除本地/远端 feature 分支
5. 清理远端跟踪：使用 `git fetch --prune`，不要手工删除不确定的分支引用

详细流程见 [references/branch-lifecycle.md](references/branch-lifecycle.md)。

---

## Tag / Release 管理

Tag 和 release 属于发布资产，默认只读分析，不主动创建/删除/覆盖。

必须确认：
1. release 版本号或 tag 命名规则
2. 目标 commit hash
3. 编译和设备/集成测试已通过
4. 用户明确批准创建或推送 tag
5. 修正远端 tag 时必须二次确认，优先保留审计记录
6. 禁止用 force push 覆盖远端 tag；如 tag 已被消费，优先创建新 tag（如 `v1.2.3-fix.1`）

详细流程见 [references/tag-release-guide.md](references/tag-release-guide.md)。

---

## Secret / 敏感信息处理

发现 token、password、private key、证书、内网地址等敏感信息时：

1. 立刻停止 commit/push/rebase 等传播动作
2. 判断是否已 push 到远端
3. 提醒用户轮换/吊销泄漏凭据
4. 未 push：从工作区或本地 commit 中移除后重新提交
5. 已 push：优先走安全响应流程；历史清理属于高风险操作，必须用户确认

详细流程见 [references/secret-and-sensitive-data.md](references/secret-and-sensitive-data.md)。

---

## 禁止操作（详细清单）

```bash
# ❌ 以下命令禁止使用：
git push --force                    # 破坏远端历史
git push -f origin main             # 绝对禁止
git push origin main                # 直接推 main，禁止
git clean -xdf                      # 删除 .gitignore 外所有文件
git reset --hard HEAD~N             # 丢失提交（需用户明确确认）
git commit --amend                  # 修改已 push 的提交禁止；未 push 时需用户确认
git tag -d <tag> && git push --delete origin <tag>  # 远端 tag 删除需二次确认；禁止 force-update tag

# ❌ Yocto 禁止操作：
bitbake -c cleansstate spcam        # 损坏 mergerfs sysroot
bitbake -c clean spcam              # 同上
```

---

## 关键检查点

```
Commit 前自查:
- [ ] 当前在 feature 分支，不在 main?
- [ ] git fetch origin 已执行，远端无新提交（或已 rebase）?
- [ ] 编译已通过?
- [ ] 用户已确认设备测试通过?
- [ ] 用户明确说"可以提交"?

Push 前自查:
- [ ] git fetch origin 已执行?
- [ ] 本地不落后于远端?
- [ ] 用户明确说"可以推送"?
```

---

## 参考文档

| 主题 | 路径 |
|------|------|
| 编译验证与设备部署 | `references/build-and-deploy.md` |
| Pull + Rebase 完整流程 | `references/pull-rebase-workflow.md` |
| 冲突解决流程 | `references/conflict-resolution.md` |
| Cherry-pick 与回退操作 | `references/cherry-pick-guide.md` |
| Git Bisect & Reflog | `references/bisect-reflog-guide.md` |
| 分支生命周期 | `references/branch-lifecycle.md` |
| Tag / Release 管理 | `references/tag-release-guide.md` |
| Secret / 敏感信息处理 | `references/secret-and-sensitive-data.md` |
