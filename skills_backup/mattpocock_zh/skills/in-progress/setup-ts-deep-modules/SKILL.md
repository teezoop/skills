---
name: setup-ts-deep-modules
description: 将 dependency-cruiser 接入 TypeScript 仓库，使每个包成为深模块：实现隐藏在子文件夹中，只能通过入口文件访问。由用户调用。
disable-model-invocation: true
---

# 配置 TypeScript 深模块

让仓库中的每个包都成为**深模块**：在小接口后封装大量行为。包的公开表面是它的**入口点**，即包根目录中的文件；所有子文件夹内容都隐藏。本技能安装 [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) 及相应规则，使入口点成为唯一访问方式，并证明规则确实能够拦截违规。

调用 Skill 工具加载 "codebase-design"，获取深模块、接口、接缝、深度等术语，并全程采用其语言。

## 强制的结构

```
src/packages/
  <name>/
    index.ts        ← 公开入口点；包外从这里导入。
    client.ts       ← 另一个入口点；一个包可以暴露多个入口。
    lib/            ← 实现；对外隐藏，内部可自由相互导入。
    tests/          ← 就近放置测试和测试数据；位于子文件夹，因此是私有内容。
```

公开表面是包的**根目录文件**，而非唯一指定的 `index.ts`。惯例是实现放在 `lib/`，测试放在 `tests/`，使所有包具有相同的双文件夹形态。但规则本身是通用的：*任何*子文件夹里的*任何*内容都是私有的，因此新增文件夹不需要扩展配置。

四条规则，严重级别均为 `error`：

1. **入口点边界**：包外代码，无论应用代码还是其他包，都只能导入该包的入口点，即根目录文件，绝不导入其子文件夹内容。
2. **包内自由**：包内文件可以自由相互导入。
3. **测试通过入口点访问**：`<pkg>/tests/` 中的文件可以导入任意包的入口点，以及本包 `tests/` 中的测试数据，但绝不能导入任何包子文件夹中的内部实现，包括本包自己的实现。跨包集成测试可以，深层导入不可以。
4. **无环**：不允许依赖环。

**是入口点，不是桶文件（barrel）。** 因为公开表面包含*每一个*根目录文件，包可以暴露多个小入口，如 `index.ts`、`client.ts`、`server.ts`，无需把所有内容汇入一个庞大的 `index.ts`。不鼓励重新导出整棵子树的桶文件；保持入口简小，将实现隐藏在子文件夹中。

分层，即哪些包可以依赖哪些包，是*另一个*问题。配置中仅以注释占位，留给本仓库自行填写。

## 步骤

### 1. 检测环境

- **包管理器**：`pnpm-lock.yaml` → pnpm，`yarn.lock` → yarn，`bun.lockb` → bun，否则 npm。下文所有命令均使用它（`pnpm` / `yarn` / `npm run` / `bunx`）。
- **包根目录**：存在 `src/` 时使用 `src/packages`，否则使用 `packages`。如果仓库已有另一套明显约定，与用户确认选择。
- **现有配置**：检查 `.dependency-cruiser.*` 文件。如果存在，**不要**覆盖，而是合并四条规则及选项，并告诉用户新增了什么。

**完成标准：** 已知包管理器、包根目录，以及是否已有配置。

### 2. 安装 dependency-cruiser

使用检测到的包管理器，将 `dependency-cruiser` 安装为 devDependency。

**完成标准：** `dependency-cruiser` 已加入 `devDependencies`。

### 3. 写入配置

将 [`dependency-cruiser.config.cjs`](./dependency-cruiser.config.cjs) 复制到仓库根目录，命名为 `.dependency-cruiser.cjs`。将 `PACKAGES_ROOT` 设置为步骤 1 检测出的目录。规则基于路径深度，与扩展名无关，因此无需其他调整。

**完成标准：** `.dependency-cruiser.cjs` 存在，`PACKAGES_ROOT` 正确，且包含四条禁止规则。

### 4. 接入检查

- 添加 `lint:boundaries` 脚本：`depcruise <packages-root>`，或 `depcruise src`。
- 将其并入仓库现有的总检查命令，即已经运行类型检查的命令，例如 `check` / `ci` / `validate`。**不要**改动 `tsconfig`，也不要添加路径别名。
- 如果没有总检查脚本，添加 `lint:boundaries`，并告诉用户将其纳入 CI。

**完成标准：** `lint:boundaries` 存在，且作为与类型检查相同的命令的一部分运行。

### 5. 搭建示例包

创建并提交一个可直接复制的 `<packages-root>/example/` 模板：

- `index.ts` 是入口点，导出一个委派给内部文件的函数，使包明显是*深模块*，而不是透传。
- `lib/impl.ts` 是位于**子文件夹**中的内部文件，由 `index.ts` 导入，包外不可访问。
- `tests/example.test.ts` **只**导入 `../index`，即入口点，并对公开函数断言。

告诉用户这是可以复制或删除的起始模板。

**完成标准：** 示例包存在，通过根目录入口暴露行为，并将 `impl` 隐藏在子文件夹中。

### 6. 证明规则有效

这是整个技能的完成标准：不能在违规时失败的配置没有价值。

1. 运行 `lint:boundaries`，它必须在干净示例上**通过**。
2. 临时在 `tests/example.test.ts` 中添加深层导入，例如 `import { thing } from "../lib/impl"`。再次运行 `lint:boundaries`，必须因 `tests-through-entrypoints` 而**失败**。
3. 撤销深层导入，再次运行，必须**通过**。

**完成标准：** 已观察到先通过、再因深层导入失败、最后再次通过。如果步骤 2 没有失败，说明规则没有正确接入，必须先修好再结束。

### 7. 记录约定

在**包目录中**编写 `README.md`（`<packages-root>/README.md`，与它约束的包并列），涵盖：`src/packages/<name>/` 布局（根目录为入口，`lib/` 放实现，`tests/` 放测试）、“只通过包的入口点，即根目录文件导入”，以及如何运行 `lint:boundaries`。明确**不鼓励桶文件**：暴露多个小入口，而不是用一个 index 重新导出整棵子树。篇幅控制为可复制的示例，加上每条规则各一段说明。

再从仓库代理指令文件添加一个**上下文指针**：优先 `CLAUDE.md`，否则 `AGENTS.md`；两者都没有时创建 `AGENTS.md`。一行即可，例如：`包是深模块：添加或导入前，参见 [src/packages/README.md](./src/packages/README.md)。` 这能让代理事先发现边界规则，而不是撞上规则后才知道。

**完成标准：** `<packages-root>/README.md` 存在且明确不鼓励桶文件，仓库的 `CLAUDE.md` / `AGENTS.md` 链接到它。

## 说明

- 配置中的 `$1` 反向引用，即 dependency-cruiser 的分组匹配，使包可以访问自己的内部，而外部不能。不要把它们展开成每包一套独立规则。
- 公开与私有由**深度**决定：包根目录文件是入口，任何子文件夹内容都是私有。约定的子文件夹是 `lib/`（实现）和 `tests/`，但规则并未硬编码它们：任意子文件夹都是私有，因此新增文件夹不需要改配置。新增入口点只是添加根目录文件，无需桶文件。
- 包是**扁平**的：只在包根目录下有一层直接子包。包内部可以任意嵌套，但一个包不能包含另一个包。
- 使用 `.cjs` 而非 `.js`，让配置中的 `module.exports` 在 `"type": "module"` 仓库中也能工作。
