---
name: kg_ue_self_commit
description: 从远端源分支 cherry-pick 最近 n 个提交或指定 hash 列表到基于 kgcode-develop 新建的目标分支并推送，推送成功后自动创建 KDev MR 并输出完整长链接。此 skill 不会自动触发，只能通过明确指定 /kg_ue_self_commit 调用。
user_invocable: true
---

# kg_ue_self_commit — 从源分支 Pick 提交到新目标分支

## 触发条件

此 skill **不会自动触发**，只能通过用户明确调用 `/kg_ue_self_commit` 使用。

## 调用形式

支持两种模式，根据第二个参数自动识别：

### 模式 1：按数量 pick（n 模式）

```
/kg_ue_self_commit <targetbranch> [n] [srcbranch]
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `targetbranch` | 是 | 目标分支名称，将在本地创建并关联到远端同名分支 |
| `n` | 否 | 要 cherry-pick 的最近提交数量，**1–99 之间的整数**，默认为 1 |
| `srcbranch` | 否 | 远端源分支名称，从该分支获取最近的提交。默认读取 `D:\c7\ue5-kdev` 仓库的当前分支名（通过 `git -C /d/c7/ue5-kdev rev-parse --abbrev-ref HEAD`） |

示例：

```
/kg_ue_self_commit my-fix
/kg_ue_self_commit my-fix 3
/kg_ue_self_commit my-fix 3 feature-x
```

### 模式 2：按 hash pick（hash 模式）

```
/kg_ue_self_commit <targetbranch> <hash[,hash...]> [srcbranch]
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `targetbranch` | 是 | 目标分支名称，将在本地创建并关联到远端同名分支 |
| `hash[,hash...]` | 是 | 一个或多个提交 hash，多个 hash 用**纯逗号**分隔，**不允许包含空格**。支持 short hash（≥4 位），由 `git rev-parse` 解析为完整 hash |
| `srcbranch` | 否 | 远端源分支名称，所有 hash 必须是 `origin/<srcbranch>` 的祖先。默认读取 `D:\c7\ue5-kdev` 仓库的当前分支名 |

示例：

```
/kg_ue_self_commit my-fix abc1234
/kg_ue_self_commit my-fix abc1234,def5678,9012345
/kg_ue_self_commit my-fix abc1234,def5678 feature-x
```

## 参数模式判定规则

第二个参数按以下规则判定模式：

- 若匹配正则 `^[0-9]{1,2}$`（即 1–2 位纯十进制数字，1–99）→ **n 模式**
- 否则 → **hash 模式**

参考 bash 判定片段：

```bash
arg2="$2"
if [[ "$arg2" =~ ^[0-9]{1,2}$ ]]; then
  mode="n"
  n="$arg2"
else
  mode="hash"
  # 按逗号切分；禁止空格
  if [[ "$arg2" == *" "* ]]; then
    echo "错误：hash 列表不允许包含空格"; exit 1
  fi
  IFS=',' read -ra hashes <<< "$arg2"
fi
```

**注意**：超出 99 的纯数字会被当作 hash 处理（几乎不会匹配到真 hash，将在校验阶段报错）。

## 完整流程

### Step 1：解析参数

从用户输入中解析出 `targetbranch`、第二个参数、`srcbranch`。

- `targetbranch` 必填，缺少则提示用户补充
- 第二个参数按上述判定规则识别为 n 模式或 hash 模式
  - n 模式下 `n` 缺省为 1
  - hash 模式下第二个参数必填
- `srcbranch` 默认通过以下命令读取 `D:\c7\ue5-kdev` 仓库的当前分支名：

  ```bash
  git -C /d/c7/ue5-kdev rev-parse --abbrev-ref HEAD
  ```

### Step 2：检查工作目录状态

```bash
git status --porcelain
```

如果有未提交的修改，提醒用户先处理（提交或 stash），然后停止执行。

### Step 3：Fetch 远端分支

```bash
git fetch origin <srcbranch> kgcode-develop
```

### Step 4：解析并校验提交列表

#### n 模式

```bash
# 查看将要 pick 的提交列表（按时间从新到旧）
git log origin/<srcbranch> --oneline -<n>
```

展示提交列表给用户，**等待 5 秒**让用户确认：

```bash
sleep 5
```

- 如果用户在 5 秒内回复确认或拒绝，按用户指示执行
- 如果 5 秒内用户没有任何回复，视为**自动同意**，继续执行后续步骤

#### hash 模式

按以下顺序对每个 hash 执行严格校验，任一失败立即中止：

1. **空段校验**：按逗号切分后，不允许出现空字符串（如 `abc,,def` 或 `abc,`）。
2. **解析为完整 hash**：

   ```bash
   full_hash=$(git rev-parse --verify "${h}^{commit}")
   ```

   解析失败或歧义（ambiguous）→ 报错中止。

3. **重复校验**：所有解析后的完整 hash 必须互不相同。
4. **srcbranch 可达性校验**：

   ```bash
   git merge-base --is-ancestor <full_hash> origin/<srcbranch>
   ```

   若返回非 0（hash 不在 `origin/<srcbranch>` 上），报错中止并提示用户检查 hash 或显式传入正确的 srcbranch。

5. **kgcode-develop 排除校验**：

   ```bash
   git merge-base --is-ancestor <full_hash> origin/kgcode-develop
   ```

   若返回 0（hash 已在 `origin/kgcode-develop` 上），报错中止并提示"hash X 已在 origin/kgcode-develop 上，无需 pick"。

校验全部通过后，按 srcbranch 的拓扑顺序（父在前、子在后）排序：

```bash
# 用 rev-list 在 srcbranch 范围内对给定 hash 进行 topo-order 排序，--reverse 后即为从早到晚
git rev-list --topo-order --reverse --no-walk <full_hash1> <full_hash2> ...
```

展示排序后的提交列表（`git log --oneline --no-walk <sorted_hashes>`），让用户能在日志中事后核对。**hash 模式不 sleep**，直接进入下一步。

### Step 5：创建本地目标分支

基于远端 `kgcode-develop` 创建本地 `targetbranch` 分支，并设置跟踪远端同名分支：

```bash
# 基于远端 kgcode-develop 创建本地分支
git checkout -b <targetbranch> origin/kgcode-develop

# 设置本地分支跟踪远端同名分支
git branch --set-upstream-to=origin/<targetbranch> <targetbranch>
```

**注意**：如果本地已存在同名分支，询问用户是否删除重建或中止操作。

### Step 6：执行 Cherry-pick

按 Step 4 得到的有序提交列表（n 模式：按时间从早到晚；hash 模式：按拓扑顺序从早到晚）依次 cherry-pick：

```bash
git cherry-pick <oldest_commit>
git cherry-pick <next_commit>
...
git cherry-pick <newest_commit>
```

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
git push -u origin <targetbranch>
```

使用 `-u` 参数确保本地分支与远端分支建立跟踪关系。

### Step 8：推送成功后自动创建 KDev MR

仅在 Step 7 推送成功后执行。用户调用本 skill 即包含创建本次提交对应 MR 的授权，无需再次询问是否创建；若用户明确要求不创建 MR，则遵从该要求。

1. 使用当前可用的浏览器工具操作 KDev 网页，复用已有登录状态，并遵守工具的浏览器操作说明。优先打开 `git push` 输出的 `KDev:` 创建链接；若输出中没有链接，从 KDev 仓库页面进入创建合并请求。不要改用 GitLab 创建页，也不要猜测内部 API。
2. 检查同仓库、同源分支、同目标分支是否已有开启的 MR；有则复用并输出其完整长链接，不重复创建。
3. 核对 MR 源分支为本次新建的 `<targetbranch>`，目标分支为 `kgcode-develop`（不是 cherry-pick 的 `<srcbranch>`）。确认页面提交和文件变更范围与本次推送一致。
4. 单提交沿用提交标题；多提交按实际整体改动概括标题并保留已有工单信息。描述说明问题、修改后的行为、主要变更和实际验证情况；未执行的编译、测试、Cook 或真机验证如实注明，不把静态检查写成运行验证通过。
5. 保留「开启检查项均通过后自动合并」和「合入完成后删除源分支」的页面默认状态，不主动关闭；其余设置也沿用页面默认值。提交后可由平台按自动合并设置处理；不手动执行核准或合并，不额外添加通知人或群通知。
6. 点击「提交合并请求」。以创建成功提示及 MR 详情页的编号、源/目标分支为依据确认结果，并核对描述和合并设置确实保存。若描述仍为默认内容，通过「编辑」补充并保存，然后重新读取页面确认。
7. 从已确认的 MR 详情页地址取得完整长链接，必须使用 `https://kdev.corp.kuaishou.com/` 下的真实详情 URL，保留页面提供的仓库路径、`mrId`、`repoId` 等参数。不要输出 `ksurl.cn` 短链接，不从 Squash Message 中取短链接，也不要把创建页链接作为结果。示例格式（参数必须来自实际页面）：

   ```text
   https://kdev.corp.kuaishou.com/git/<namespace>/<project>/-/merge_requests?mrId=<实际MR编号>&repoId=<实际仓库编号>&tabKey=baseInfo
   ```

**异常处理**：提交或编辑接口超时时，先刷新详情页或检查 MR 列表与活动记录，判断是否已经成功，再决定是否重试，避免重复创建。确认未成功后最多重试一次；仍失败或浏览器不可用、需要用户登录时，报告具体阻塞并保留已推送分支，不声称 MR 已创建。即使 MR 步骤受阻，仍执行后续切回及快进更新步骤，并分别报告 Git 与 MR 的实际状态。

### Step 9：切换回 kgcode-develop 分支并更新到最新

推送完成后，将当前工作目录切换回本地 `kgcode-develop` 分支，并将其更新到远端最新：

```bash
git checkout kgcode-develop
git pull --ff-only origin kgcode-develop
```

如果本地不存在 `kgcode-develop` 分支，则基于远端创建（已自动跟踪远端最新）：

```bash
git checkout -b kgcode-develop origin/kgcode-develop
```

**注意**：使用 `--ff-only` 确保仅在能快进合并时更新，若本地分支与远端存在分歧则停止并提示用户处理。

### Step 10：输出结果摘要

完成后输出摘要信息（`已 pick 的提交` 按实际 pick 顺序，即 topo-order 从早到晚展示）：

```
✅ Cherry-pick 完成

- 模式：<n 模式 | hash 模式>
- 源分支：origin/<srcbranch>
- 目标分支：<targetbranch>（基于 origin/kgcode-develop 创建）
- 已 pick 的提交（共 N 个，按 pick 顺序）：
  - <hash1> <描述1>
  - <hash2> <描述2>
  ...
- 远端推送：已完成（跟踪 origin/<targetbranch>）
- KDev MR：<已创建或复用的 MR 编号>
- MR 完整长链接：<从详情页获取的 https://kdev.corp.kuaishou.com/... URL，禁止使用短链接>
- 当前分支：已切换回 kgcode-develop
```

摘要按实际执行结果填写；MR 未创建时明确原因，不填入虚构链接。成功创建或复用 MR 后，最终回复必须自动包含可点击的完整长链接。

## 注意事项

1. **参数模式**：第二个参数为 1–99 的纯整数时按 n 模式处理；否则按 hash 模式（hash 列表用逗号分隔，**不允许空格**）。
2. **提交顺序**：
   - n 模式：按时间从早到晚 cherry-pick
   - hash 模式：按 srcbranch 上的拓扑顺序（`git rev-list --topo-order --reverse`）从早到晚 cherry-pick
3. **hash 校验严格**：空段、重复 hash、不在 `origin/<srcbranch>` 上、已在 `origin/kgcode-develop` 上 — 任一情况立即中止报错。
4. **冲突处理**：遇到冲突时必须暂停并通知用户，不要自动解决冲突。
5. **确认机制**：n 模式展示提交列表后等待 5 秒，超时未回复则自动同意；hash 模式展示但不等待，直接继续。
6. **分支保护**：操作前确认当前工作目录没有未提交的修改。
7. **回退方案**：如果 push 失败，提示用户可能的原因并建议解决方案。
8. **分支冲突**：如果本地已存在同名目标分支，必须询问用户如何处理。
