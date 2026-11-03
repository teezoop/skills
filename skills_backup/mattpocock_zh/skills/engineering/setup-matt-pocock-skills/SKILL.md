---
name: setup-matt-pocock-skills
description: 为仓库配置工程技能，包括问题跟踪系统、分诊标签术语和领域文档布局。首次使用其他工程技能前运行一次。
disable-model-invocation: true
---

# 配置 Matt Pocock 的技能

建立工程技能默认需要的仓库级配置：

- **问题跟踪系统**：工单存放在哪里，默认 GitHub，也原生支持本地 Markdown。
- **分诊标签**：五个规范分诊角色使用的标签字符串。
- **领域文档**：`GLOSSARY.md` 和 ADR 的位置，以及使用方如何读取它们。

这是由提示词驱动的技能，而非确定性脚本。先探索，再展示发现，与用户确认后写入。

## 流程

### 1. 探索

查看当前仓库，了解起始状态。读取实际存在的内容，不要假设：

- `git remote -v` 和 `.git/config`：是否为 GitHub 仓库？具体是哪一个？
- 仓库根目录的 `AGENTS.md` 和 `CLAUDE.md`：是否存在？是否已有 `## Agent skills` 小节？
- 根目录的 `GLOSSARY.md`、`GLOSSARY-MAP.md`，以及旧版 `CONTEXT.md` / `CONTEXT-MAP.md`。如果存在旧版文件，提议使用 `git mv` 改为对应的 `GLOSSARY` 名称，并更新 `docs/agents/domain.md` 中的名称。
- `docs/adr/` 和所有 `src/*/docs/adr/` 目录。
- `docs/agents/`：是否已有本技能之前的输出？
- `.scratch/`：这可能表明仓库已采用本地 Markdown 问题跟踪约定。
- 是否安装 `triage` 技能？检查本技能旁边的 `triage` 文件夹，或可用技能列表中的 `triage`。这决定是否执行 B 部分。
- Monorepo 特征：`pnpm-workspace.yaml`、`package.json` 中的 `workspaces` 字段，或 `packages/*` 中存在实际内容且具有各自 `src/` 的包。这些只会出现在真正的大型多包仓库；没有这些信号，就采用单上下文，几乎所有仓库都是如此。

### 2. 展示发现并询问

总结现有内容与缺失内容。随后按顺序处理各部分：一次一部分，得到一个回答，再进入下一部分。

每部分先给出推荐答案，使用户一句话即可接受。只有选项确实会产生分支时，才给一句解释。探索已经确定的部分应直接跳过，例如未安装 `triage` 时跳过 B 部分，没有 monorepo 时跳过 C 部分的提问。

**A 部分：问题跟踪系统。**

> 说明：“问题跟踪系统”是此仓库存放工单的位置。`to-tickets`、`triage` 和 `to-spec` 等技能会读写它。它们需要知道应调用 `gh issue create`、在 `.scratch/` 下写 Markdown 文件，还是遵循你描述的其他流程。选择你实际用于跟踪该仓库工作的位置。

默认立场：这些技能原本面向 GitHub 设计。如果某个 `git remote` 指向 GitHub，就推荐 GitHub；如果指向 GitLab，包括 `gitlab.com` 或自托管主机，就推荐 GitLab。否则，或用户有其他偏好时，提供以下选择：

- **GitHub**：工单位于仓库的 GitHub Issues，使用 `gh` CLI。
- **GitLab**：工单位于仓库的 GitLab Issues，使用 [`glab`](https://gitlab.com/gitlab-org/cli) CLI。
- **本地 Markdown**：工单作为文件存放在仓库的 `.scratch/<feature>/` 下，适合个人项目或没有远程仓库的项目。
- **其他**（Jira、Linear 等）：让用户用一段话描述流程，本技能将其记录为自由形式的文字。

将选择记录到 `docs/agents/issue-tracker.md`。GitHub 和 GitLab 模板具有“将 PR 作为请求入口”的开关，默认**关闭**。保持关闭，也不要主动提起；希望把外部 PR 放进分诊队列的用户可以稍后自行修改该文件。

GitHub 或 GitLab 跟踪系统需要可访问的仓库。用户选定后，如果没有 `git remote` 指向相应主机，向用户获取仓库 URL，并在继续前配置好：必要时 `git init`，然后 `git remote add origin <url>`。在当前目录运行 `gh repo view` / `glab repo view` 能显示该项目后才算完成；如果无法做到，停止并说明缺少什么。

**B 部分：分诊标签术语。** 如果未安装 `triage` 技能，完全跳过本节；探索阶段已经查明，未安装的技能不需要标签。

已安装时，只问一个问题：

> 是否保留默认的分诊标签？（推荐：**是**）

默认值为五个规范角色，标签字符串与角色名一致：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。回答**是**时，原样写入。只有用户回答否时，才收集覆盖值，通常是其跟踪系统已有其他名称，例如用 `bug:triage` 对应 `needs-triage`，让 `triage` 使用现有标签而非创建重复项。

**C 部分：领域文档。** 默认采用**单上下文**，在仓库根目录使用一个 `GLOSSARY.md` 加 `docs/adr/`。这适合几乎所有仓库，无需询问，直接写入。

只有探索发现 monorepo 特征时，才提供**多上下文**方案，即根目录的 `GLOSSARY-MAP.md` 指向各上下文的 `GLOSSARY.md`，然后确认用户想要哪种布局。

### 3. 确认与编辑

向用户展示以下草稿：

- 准备加入所选 `CLAUDE.md` / `AGENTS.md` 的 `## Agent skills` 块，选择规则见步骤 4。
- `docs/agents/issue-tracker.md`、`docs/agents/domain.md` 和 `docs/agents/triage-labels.md` 的内容；最后一项仅在已安装 `triage` 时提供。

写入前允许用户修改。

### 4. 写入

**选择要编辑的文件：**

- 存在 `CLAUDE.md` 时，编辑它。
- 否则，存在 `AGENTS.md` 时，编辑它。
- 两者都不存在时，询问用户创建哪一个，不要替用户选择。

已有 `CLAUDE.md` 时，绝不创建 `AGENTS.md`，反之亦然；始终编辑现有文件。

如果所选文件已有 `## Agent skills` 块，就原位更新，而非追加重复块。不要覆盖用户对周围小节的修改。

内容块：

```markdown
## Agent skills

### Issue tracker

[用一句话说明工单在哪里跟踪]。参见 `docs/agents/issue-tracker.md`。

### Triage labels

[用一句话说明标签术语]。参见 `docs/agents/triage-labels.md`。

### Domain docs

[用一句话说明布局：“single-context”或“multi-context”]。参见 `docs/agents/domain.md`。
```

只有安装了 `triage` 且执行了 B 部分时，才包含 `### Triage labels` 子块，并写入 `docs/agents/triage-labels.md`。否则两者都省略。

在 GitHub 或 GitLab 上执行 B 部分时，创建跟踪系统缺少的每个已配置标签，使用 `gh label create` / `glab label create`。

随后以本技能文件夹中的初始模板为起点，编写文档：

- [issue-tracker-github.md](./issue-tracker-github.md)：GitHub 问题跟踪系统。
- [issue-tracker-gitlab.md](./issue-tracker-gitlab.md)：GitLab 问题跟踪系统。
- [issue-tracker-local.md](./issue-tracker-local.md)：本地 Markdown 问题跟踪系统。
- [triage-labels.md](./triage-labels.md)：标签映射，仅在安装 `triage` 时需要。
- [domain.md](./domain.md)：领域文档使用规则与布局。

对于“其他”问题跟踪系统，根据用户描述从头编写 `docs/agents/issue-tracker.md`。

### 5. 完成

告诉用户配置已完成，以及哪些工程技能接下来会读取这些文件。说明稍后可以直接编辑 `docs/agents/*.md`；只有想切换问题跟踪系统或从头配置时，才需要重新运行本技能。
