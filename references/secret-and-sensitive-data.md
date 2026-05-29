# Secret / 敏感信息处理指南

## 敏感信息类型

- API token / access key / password
- SSH private key / TLS private key
- 证书、cookie、session
- 内网地址、客户数据、设备序列号（按项目保密规则判断）

## 第一原则：先止血，再改历史

发现敏感信息后，不要继续 commit、push、rebase 或复制日志。

必须先确认：

1. 是否已经 push 到远端。
2. 是否已经进入 CI/artifact/release。
3. 凭据是否需要立刻轮换或吊销。

## 未 push 的处理

### 还在工作区

```bash
git status
git diff -- <file>
```

从文件中移除 secret，加入 `.gitignore` 或改用环境变量/配置模板。

### 已 commit 但未 push

```bash
git log --oneline -5
git show HEAD -- <file>
```

用户确认后可使用 amend 或 interactive rebase 修正本地历史。

## 已 push 的处理

已 push 后不能只从最新 commit 删除；历史中仍可见。

安全流程：

1. 立刻轮换/吊销泄漏凭据。
2. 通知相关负责人。
3. 评估是否需要清理 Git 历史。
4. 历史清理需团队确认，因为会影响所有协作者。

可选工具（高风险，不能自动执行）：

```bash
git filter-repo --path <file> --invert-paths
```

或使用 BFG Repo-Cleaner。执行后通常需要 force push，因此必须单独审批。

## 检查命令

```bash
git grep -nE "(AKIA|token|password|secret|private_key|BEGIN RSA PRIVATE KEY)"
git log -p --all -- <file>
```

## 防止再次发生

- `.env`、私钥、生成凭据加入 `.gitignore`。
- 提供 `.env.example`，不提交真实值。
- 使用 pre-commit secret scanning。
- 日志和评测结果中避免打印 token。