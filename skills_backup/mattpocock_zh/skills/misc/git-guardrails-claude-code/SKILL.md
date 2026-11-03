---
name: git-guardrails-claude-code
description: 配置 Claude Code 钩子，在危险的 Git 命令（push、reset --hard、clean、branch -D 等）执行前将其拦截。适用于用户希望防止破坏性 Git 操作、添加 Git 安全钩子，或在 Claude Code 中阻止 git push/reset 的情况。
---

# 配置 Git 防护

配置一个 PreToolUse 钩子，在 Claude 执行危险的 Git 命令之前进行拦截。

## 拦截范围

- `git push`（所有变体，包括 `--force`）
- `git reset --hard`
- `git clean -f` / `git clean -fd`
- `git branch -D`
- `git checkout .` / `git restore .`

命令被拦截后，Claude 会收到一条消息，告知它无权使用这些命令。

## 步骤

### 1. 询问作用范围

询问用户：仅为**当前项目**安装（`.claude/settings.json`），还是为**所有项目**安装（`~/.claude/settings.json`）？

### 2. 复制钩子脚本

随附脚本位于：[scripts/block-dangerous-git.sh](scripts/block-dangerous-git.sh)

根据作用范围复制到目标位置：

- **项目级**：`.claude/hooks/block-dangerous-git.sh`
- **全局**：`~/.claude/hooks/block-dangerous-git.sh`

使用 `chmod +x` 添加可执行权限。

### 3. 将钩子加入设置

添加到相应的设置文件中：

**项目级**（`.claude/settings.json`）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous-git.sh"
          }
        ]
      }
    ]
  }
}
```

**全局**（`~/.claude/settings.json`）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-dangerous-git.sh"
          }
        ]
      }
    ]
  }
}
```

如果设置文件已存在，将钩子合并到现有的 `hooks.PreToolUse` 数组中。不要覆盖其他设置。

### 4. 询问是否需要定制

询问用户是否希望增删拦截列表中的匹配规则，并相应修改复制后的脚本。

### 5. 验证

运行一个快速测试：

```bash
echo '{"tool_input":{"command":"git push origin main"}}' | <path-to-script>
```

应以退出码 2 结束，并向标准错误输出一条 BLOCKED 消息。
