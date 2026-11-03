---
name: setup-pre-commit
description: 在当前仓库配置 Husky pre-commit 钩子，结合 lint-staged（Prettier）、类型检查和测试。适用于用户希望添加 pre-commit 钩子、配置 Husky 或 lint-staged，或在提交时执行格式化、类型检查及测试的情况。
---

# 配置 Pre-Commit 钩子

## 将配置的内容

- **Husky** pre-commit 钩子。
- **lint-staged**，对所有暂存文件运行 Prettier。
- **Prettier** 配置（如果缺失）。
- 在 pre-commit 钩子中运行 **typecheck** 和 **test** 脚本。

## 步骤

### 1. 检测包管理器

检查 `package-lock.json`（npm）、`pnpm-lock.yaml`（pnpm）、`yarn.lock`（yarn）和 `bun.lockb`（bun）。使用已有锁文件对应的包管理器；不明确时默认使用 npm。

### 2. 安装依赖

将以下依赖安装为 devDependencies：

```
husky lint-staged prettier
```

### 3. 初始化 Husky

```bash
npx husky init
```

这会创建 `.husky/` 目录，并在 package.json 中添加 `prepare: "husky"`。

### 4. 创建 `.husky/pre-commit`

写入以下内容（Husky v9 及以上不需要 shebang）：

```
npx lint-staged
npm run typecheck
npm run test
```

**根据项目调整**：将 `npm` 替换为检测到的包管理器。如果仓库的 package.json 中没有 `typecheck` 或 `test` 脚本，就省略相应行，并告知用户。

### 5. 创建 `.lintstagedrc`

```json
{
  "*": "prettier --ignore-unknown --write"
}
```

### 6. 创建 `.prettierrc`（如果缺失）

仅在尚无 Prettier 配置时创建，使用以下默认值：

```json
{
  "useTabs": false,
  "tabWidth": 2,
  "printWidth": 80,
  "singleQuote": false,
  "trailingComma": "es5",
  "semi": true,
  "arrowParens": "always"
}
```

### 7. 验证

- [ ] `.husky/pre-commit` 存在且可执行。
- [ ] `.lintstagedrc` 存在。
- [ ] package.json 中的 `prepare` 脚本为 `"husky"`。
- [ ] `prettier` 配置存在。
- [ ] 运行 `npx lint-staged`，验证其可以工作。

### 8. 提交

暂存所有修改或创建的文件，并使用以下消息提交：`Add pre-commit hooks (husky + lint-staged + prettier)`。

本次提交会运行新配置的 pre-commit 钩子，正好作为验证整体是否正常工作的冒烟测试。

## 说明

- Husky v9 及以上的钩子文件不需要 shebang。
- `prettier --ignore-unknown` 会跳过 Prettier 无法解析的文件，例如图片。
- pre-commit 先运行 lint-staged（速度快，仅处理暂存文件），再运行完整的类型检查和测试。
