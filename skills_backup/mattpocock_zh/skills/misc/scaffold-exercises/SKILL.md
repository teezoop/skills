---
name: scaffold-exercises
description: 创建含课程章节、题目、解答和讲解的练习目录结构，并使其通过 lint 检查。适用于用户希望搭建练习骨架、创建练习占位文件，或建立新课程章节的情况。
---

# 搭建练习骨架

创建能够通过 `pnpm ai-hero-cli internal lint` 的练习目录结构，然后使用 `git commit` 提交。

## 目录命名

- **章节**：位于 `exercises/` 内，格式为 `XX-section-name/`，例如 `01-retrieval-skill-building`。
- **练习**：位于某一章节内，格式为 `XX.YY-exercise-name/`，例如 `01.03-retrieval-with-bm25`。
- 章节编号为 `XX`，练习编号为 `XX.YY`。
- 名称使用小写字母和连字符，即 dash-case。

## 练习类型

每个练习至少需要以下子文件夹之一：

- `problem/`：含 TODO 的学生工作区。
- `solution/`：参考实现。
- `explainer/`：概念讲解材料，不含 TODO。

搭建占位骨架时，除非计划另有说明，否则默认使用 `explainer/`。

## 必需文件

每个子文件夹（`problem/`、`solution/`、`explainer/`）都需要一个 `readme.md`，且满足：

- **不为空**，必须有实际内容，哪怕只有一行标题也可以。
- 不含失效链接。

搭建骨架时，创建一个包含标题和说明的最小 readme：

```md
# 练习标题

在这里填写说明
```

如果子文件夹中有代码，还需要一个超过 1 行的 `main.ts`。但对于占位骨架，只有 readme 的练习也可以。

## 工作流程

1. **解析计划**：提取章节名、练习名和类型。
2. **创建目录**：为每条路径执行 `mkdir -p`。
3. **创建 readme 占位文件**：为每个类型文件夹创建一个带标题的 `readme.md`。
4. **运行 lint**：执行 `pnpm ai-hero-cli internal lint` 进行验证。
5. **修复错误**：迭代直到 lint 通过。

## lint 规则摘要

检查器（`pnpm ai-hero-cli internal lint`）检查以下内容：

- 每个练习具有子文件夹（`problem/`、`solution/`、`explainer/`）。
- 至少存在 `problem/`、`explainer/` 或 `explainer.1/` 中的一个。
- 主要子文件夹中存在非空的 `readme.md`。
- 没有 `.gitkeep` 文件。
- 没有 `speaker-notes.md` 文件。
- readme 中没有失效链接。
- readme 中没有 `pnpm run exercise` 命令。
- 除非该子文件夹只有 readme，否则必须有 `main.ts`。

## 移动或重命名练习

重新编号或移动练习时：

1. 使用 `git mv`（而不是 `mv`）重命名目录，以保留 Git 历史。
2. 更新数字前缀，保持顺序。
3. 移动后重新运行 lint。

示例：

```bash
git mv exercises/01-retrieval/01.03-embeddings exercises/01-retrieval/01.04-embeddings
```

## 示例：根据计划搭建骨架

假设计划如下：

```
第 05 章：记忆技能构建
- 05.01 记忆简介
- 05.02 短期记忆（explainer + problem + solution）
- 05.03 长期记忆
```

创建：

```bash
mkdir -p exercises/05-memory-skill-building/05.01-introduction-to-memory/explainer
mkdir -p exercises/05-memory-skill-building/05.02-short-term-memory/{explainer,problem,solution}
mkdir -p exercises/05-memory-skill-building/05.03-long-term-memory/explainer
```

随后创建 readme 占位文件：

```
exercises/05-memory-skill-building/05.01-introduction-to-memory/explainer/readme.md -> "# 记忆简介"
exercises/05-memory-skill-building/05.02-short-term-memory/explainer/readme.md -> "# 短期记忆"
exercises/05-memory-skill-building/05.02-short-term-memory/problem/readme.md -> "# 短期记忆"
exercises/05-memory-skill-building/05.02-short-term-memory/solution/readme.md -> "# 短期记忆"
exercises/05-memory-skill-building/05.03-long-term-memory/explainer/readme.md -> "# 长期记忆"
```
