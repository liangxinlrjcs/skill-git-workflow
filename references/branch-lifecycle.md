# 分支生命周期管理

## 目标

减少长期分支漂移、误删远端分支、错误基线开发和 rebase 冲突风险。

## 标准生命周期

| 阶段 | 操作 | 必须确认 |
|---|---|---|
| 创建前 | `git fetch origin`，确认基线分支 | 当前不在 main 上直接开发 |
| 创建 | `feature/MMDD-HHMM` 或 `hotfix/MMDD-HHMM` | 分支名符合规范 |
| 开发中 | 小步修改，定期查看远端差异 | 不混入无关文件 |
| 同步 | stash → fetch → rebase → stash pop | 冲突需与用户协同 |
| 合并前 | 编译 + 设备/集成测试 + review | 用户确认测试通过 |
| 合并后 | 删除本地/远端 feature 分支 | 确认已合并且无人依赖 |

## 常见场景

### 在错误分支上开发了

1. 停止当前修改，不要 commit 到错误分支。
2. 查看改动范围：`git status`、`git diff --stat`。
3. 新建正确分支：`git checkout -b feature/MMDD-HHMM`。
4. 确认改动随工作区带到新分支。
5. 编译验证后再按正常流程提交。

### 清理已删除的远端分支引用

```bash
git fetch --prune origin
git branch -r
```

只清理 remote-tracking reference，不会删除远端真实分支。

### 删除 feature 分支

```bash
git branch --merged
git branch -d feature/MMDD-HHMM
```

远端删除必须确认：

```bash
git push origin --delete feature/MMDD-HHMM
```

## 禁止事项

- 不确认合并状态就删除远端分支。
- 对多人共享分支擅自 rebase 后强推。
- 直接在 main/master 上开发并 push。
- 用 `git branch -D` 强删仍未合并的分支，除非用户明确确认。