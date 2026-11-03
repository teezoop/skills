---
name: migrate-to-shoehorn
description: 将测试文件中的 `as` 类型断言迁移到 @total-typescript/shoehorn。适用于用户提到 shoehorn、希望替换测试中的 `as`，或需要构造部分测试数据的情况。
---

# 迁移到 Shoehorn

## 为什么使用 shoehorn？

`shoehorn` 允许在测试中传入部分数据，同时满足 TypeScript 的类型检查。它用类型安全的替代方案取代 `as` 断言。

**仅用于测试代码。** 绝不要在生产代码中使用 shoehorn。

在测试中使用 `as` 的问题：

- 通常已被教导不要使用它。
- 必须手动指定目标类型。
- 为了故意传入错误数据，需要双重断言（`as unknown as Type`）。

## 安装

```bash
npm i @total-typescript/shoehorn
```

## 迁移模式

### 只需要少量属性的大对象

迁移前：

```ts
type Request = {
  body: { id: string };
  headers: Record<string, string>;
  cookies: Record<string, string>;
  // ……另有 20 个属性
};

it("gets user by id", () => {
  // 只关心 body.id，却必须伪造整个 Request
  getUser({
    body: { id: "123" },
    headers: {},
    cookies: {},
    // ……伪造全部 20 个属性
  });
});
```

迁移后：

```ts
import { fromPartial } from "@total-typescript/shoehorn";

it("gets user by id", () => {
  getUser(
    fromPartial({
      body: { id: "123" },
    }),
  );
});
```

### `as Type` → `fromPartial()`

迁移前：

```ts
getUser({ body: { id: "123" } } as Request);
```

迁移后：

```ts
import { fromPartial } from "@total-typescript/shoehorn";

getUser(fromPartial({ body: { id: "123" } }));
```

### `as unknown as Type` → `fromAny()`

迁移前：

```ts
getUser({ body: { id: 123 } } as unknown as Request); // 故意使用错误类型
```

迁移后：

```ts
import { fromAny } from "@total-typescript/shoehorn";

getUser(fromAny({ body: { id: 123 } }));
```

## 各函数的使用场景

| 函数 | 使用场景 |
| --- | --- |
| `fromPartial()` | 传入仍可通过类型检查的部分数据 |
| `fromAny()` | 故意传入错误数据，同时保留自动补全 |
| `fromExact()` | 强制提供完整对象，之后可替换为 fromPartial |

## 工作流程

1. **收集需求**，询问用户：
   - 哪些测试文件中的 `as` 断言带来了问题？
   - 是否在处理只有部分属性有用的大对象？
   - 是否需要故意传入错误数据来测试错误处理？

2. **安装并迁移**：
   - [ ] 安装：`npm i @total-typescript/shoehorn`
   - [ ] 查找含有 `as` 断言的测试文件：`grep -r " as [A-Z]" --include="*.test.ts" --include="*.spec.ts"`
   - [ ] 将 `as Type` 替换为 `fromPartial()`
   - [ ] 将 `as unknown as Type` 替换为 `fromAny()`
   - [ ] 添加来自 `@total-typescript/shoehorn` 的导入
   - [ ] 运行类型检查进行验证
