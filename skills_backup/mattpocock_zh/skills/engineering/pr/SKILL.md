---
name: pr
description: 编写 PR 正文时使用。
metadata:
  credits:
    skill: show-me
    author: Dex Horthy
    organisation: Humanlayer
    url: "https://github.com/humanlayer/skills/blob/main/plugins/show-me/skills/show-me/SKILL.md"
---

使用以下模板编写 PR 正文：

```markdown
## 摘要

<图示、差异草图或树状图>

## 证据

- **修改前：** <截图、输出或失败的测试运行>
  **修改后：** <截图、输出或通过的测试运行>

## 合并风险

**决策门类型：** <单向门或双向门>

<可选：说明>

**影响范围：** <用一个词描述>

<可选：合并可能带来的后果>
```

## 各部分说明

省略所有开场白，保持文字简洁。采用 `GLOSSARY.md` 中用户的领域语言。

### 摘要

选择能够清晰表达关键点的最小视图。

- 用伪代码展示逻辑或算法：

```text
on(save)
  if content is unchanged
    return cached result
  write new content
  return fresh result
```

- 用调用树展示运行时控制流：

```text
submitForm
  createSession
    persistPrompt
    launchAgent
  navigateToSession
```

- 用组件树展示 UI 结构，并包含重要的状态和模块边界：

```text
<SessionPage> (apps/example/src/routes/session.tsx)
  useSessionEvents()
  <SessionToolbar>
    <RunSkillButton> (packages/ui)
```

- 用浅层文件树展示文件职责或范围较大的重构：

```text
src/
├── commands/       # 解析用户操作
├── sessions/       # 管理会话状态
└── transport/      # 发送 API 请求
```

- 用 Mermaid 展示组件交互、控制流或数据流：

```mermaid
sequenceDiagram
    participant User
    participant UI
    participant Daemon
    User->>UI: 选择命令
    UI->>Daemon: 发送展开后的提示词
    Daemon-->>UI: 流式返回结果
```

- 如果重点是改变了什么，且周边结构已存在，就使用 `diff`。根据主题选择差异图的形式。

组件变更示例：

```diff
 <SessionPage>
   useSessionEvents()
   <SessionToolbar>
+    <RunSkillButton />
   <SessionTimeline>
+    <SkillResultCard />
```

文件布局变更示例：

```diff
 src/
 ├── commands/
+│   └── show-me.ts       # 展开斜杠命令
 ├── sessions/
-└── transport.ts
+└── transport/
+    ├── client.ts
+    └── stream.ts
```

调用树或调用栈变更示例：

```diff
 submitForm
   createSession
     persistPrompt
+    expandSkillMention
     launchAgent
-  navigateToSession
+  navigateToSession
+    subscribeToEvents
```

状态或控制流变更示例：

```diff
 on(save)
-  write content
+  if content is unchanged
+    return cached result
+  write new content
+  invalidate cache
```

- 如果大部分内容都是新增的、省略上下文会隐藏归属或执行顺序，或用户需要可直接复制的目标结构，就展示完整代码块：

```ts
function expandSkill(command: string): string {
  const skillName = command.slice(1);
  return `use the ${skillName} skill`;
}
```

#### 使用建议

将每个图示放在其支持的简短文字旁边。只保留回答用户当前问题、或帮助解决当前讨论点所必需的调用、文件、属性、状态和边界。

可以使用其中一种，也可以使用多种；通常不需要全部使用。自行判断，避免让用户被信息淹没。

### 证据

给出变更确实有效的具体证据，展示修改前后的对比。

在环境已就绪且变更涉及视觉效果时，截图是最高等级的证据。

基于实际执行的证据次之，例如测试结果和控制台输出。用伪代码展示具体哪个测试现在从失败变为通过。

### 合并风险

说明这是单向门还是双向门决策。双向门可以退回，单向门则无法轻易退回。容易回滚的 PR 风险较低。涉及破坏性操作或难以逆转决策的变更属于单向门。

影响范围是此 PR 引入的变更可能造成的影响或波及范围。应考虑各种可能性，例如布局偏移、对使用方造成破坏、移动端响应式表现等。
