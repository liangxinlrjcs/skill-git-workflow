# Tag / Release 管理指南

## 核心原则

Tag 是发布审计点。默认只读分析，不主动创建、删除、覆盖远端 tag。

## 创建 tag 前检查

1. 确认版本号和命名规范。
2. 确认目标 commit hash。
3. 确认编译通过。
4. 确认设备/集成测试通过。
5. 确认 changelog/release note 已准备。
6. 等待用户明确批准。

## 命名建议

| 场景 | 示例 | 说明 |
|---|---|---|
| 语义版本 | `v1.2.3` | 推荐正式 release |
| 日期版本 | `release/20260529` | 适合内部日构建 |
| Hotfix | `v1.2.4-hotfix.1` | 适合补丁发布 |

具体项目若已有规范，以项目规范为准。

## Annotated tag 优先

```bash
git tag -a v1.2.3 <commit> -m "Release v1.2.3"
git show v1.2.3
```

Annotated tag 包含 tagger、时间和说明，更适合 release 审计。

## 推送 tag

```bash
git push origin v1.2.3
```

推送前必须再次确认目标 commit：

```bash
git rev-parse v1.2.3
git log -1 --oneline v1.2.3
```

## 修正错误 tag

远端 tag 修正是高风险操作，必须二次确认。

推荐流程：

1. 说明当前 tag 指向和正确 commit。
2. 确认是否已有 CI/release/用户依赖该 tag。
3. 优先创建新 tag 或补丁 tag，避免删除历史。
4. 如必须删除远端 tag，等待用户明确批准。

禁止事项：

- 不要建议 `git push --force` 覆盖远端 tag。
- 不要用同名 tag 静默替换已经发布的 release。
- 已被 CI/release/用户消费的 tag，优先新建修正 tag（如 `v1.2.3-fix.1`）。

高风险命令示例（不要自动执行）：

```bash
git tag -d v1.2.3
git push origin --delete v1.2.3
git tag -a v1.2.3 <correct-commit> -m "Release v1.2.3"
git push origin v1.2.3
```

## Release 分支

```bash
git checkout main
git fetch origin
git checkout -b release/1.2.x origin/main
```

Release 分支上的修复应走 cherry-pick 或 PR 流程，不直接 push main。