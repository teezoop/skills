---
name: kgcode_ue_pick_to_release
description: 将指定的 git 提交从远端 kgcode-develop 分支 cherry-pick 到本地 kgcode-release-c7 分支并推送到远端。支持输入一个或多个 commit hash，也支持通过提交描述信息模糊匹配。触发关键词："kgcode pick to release"、"pick to release"。
---

# kgcode_ue_pick_to_release — Cherry-pick 提交到发布分支

## 触发条件

用户输入包含以下关键词时触发：
- `kgcode pick to release`
- `pick to release`

## 输入格式

用户可以提供以下类型的输入：

1. **一个或多个 commit hash**：如 `abc1234`、`abc1234 def5678 ghi9012`
2. **提交描述信息**：如 `"修复渲染崩溃问题"`，技能将在 `origin/kgcode-develop` 分支上搜索匹配的提交

## 完整流程

### Step 1：解析用户输入

判断用户输入的是 commit hash 还是提交描述信息：
- **commit hash**：匹配 `[0-9a-f]{7,40}` 格式的字符串，可能有多个（空格或逗号分隔）
- **提交描述信息**：非 hash 格式的文本，需要到 `origin/kgcode-develop` 上搜索匹配的提交

### Step 2：Fetch 远端最新修改

```bash
git fetch origin kgcode-develop kgcode-release-c7
```

### Step 3：确保本地 kgcode-release-c7 分支存在

```bash
# 检查本地分支是否存在
git branch --list kgcode-release-c7
```

- **如果不存在**：基于远端分支创建本地分支
  ```bash
  git checkout -b kgcode-release-c7 origin/kgcode-release-c7
  ```
- **如果已存在**：切换到该分支
  ```bash
  git checkout kgcode-release-c7
  ```

### Step 4：Pull 更新本地 kgcode-release-c7 分支

```bash
git pull origin kgcode-release-c7
```

### Step 5：（如果输入是描述信息）搜索匹配的提交

```bash
# 在 origin/kgcode-develop 上搜索包含关键词的提交
git log origin/kgcode-develop --oneline --grep="<用户输入的描述关键词>" -20
```

搜索结果展示给用户，让用户确认要 pick 哪些提交。如果只匹配到一个提交，也要和用户确认。

**重要**：如果搜索不到结果，尝试以下策略：
- 拆分关键词分别搜索
- 使用 `--all-match` 结合多个 `--grep` 参数
- 提示用户换用更精确的关键词或直接提供 commit hash

### Step 6：执行 Cherry-pick

对每个 commit 依次执行 cherry-pick：

```bash
git cherry-pick <commit_hash>
```

**多个 commit 的情况**：按提交时间从早到晚的顺序依次 cherry-pick。

**冲突处理**：
- 如果 cherry-pick 产生冲突，立即通知用户冲突的文件列表
- 展示冲突内容，协助用户解决冲突
- 冲突解决后执行：
  ```bash
  git add <resolved_files>
  git cherry-pick --continue
  ```
- 如果用户决定放弃，执行：
  ```bash
  git cherry-pick --abort
  ```

### Step 7：Push 到远端

```bash
git push origin kgcode-release-c7
```

### Step 8：输出结果摘要

完成后输出摘要信息：

```
✅ Cherry-pick 完成

- 源分支：origin/kgcode-develop
- 目标分支：kgcode-release-c7
- 已 pick 的提交：
  - <hash1> <描述1>
  - <hash2> <描述2>
  ...
- 远端推送：已完成
```

## 注意事项

1. **提交顺序**：多个 commit 按时间从早到晚的顺序 cherry-pick，避免依赖顺序问题
2. **冲突处理**：遇到冲突时必须暂停并通知用户，不要自动解决冲突
3. **确认机制**：通过描述搜索到的提交必须经用户确认后才执行 pick
4. **分支保护**：操作前确认当前工作目录没有未提交的修改，如有则提醒用户先处理
5. **回退方案**：如果 push 失败，提示用户可能的原因（如分支保护规则）并建议解决方案
