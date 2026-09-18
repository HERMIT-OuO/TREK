# 共享契约、类型与构建导出

## 按领域拥有线协议

**当前实现**：`shared/src/index.ts` 导出各领域 Schema 和 `z.infer` 类型。`shared/src/common/primitives.schema.ts` 提供：

```ts
export const idSchema = z.number().int().positive();
export const idParamSchema = z.coerce.number().int().positive();
export type Id = z.infer<typeof idSchema>;
```

`idSchema` 是 JSON 数值 ID，`idParamSchema` 用于 URL 字符串边界；客户端离线负 ID 是本地状态，不应强行通过正 ID 的服务端契约。

**新增代码约束**（`shared/CLAUDE.md`）：Schema 与推导类型共同定义；领域独立文件夹，导出接入 root barrel；新增 Schema 同时添加相邻 `.spec.ts`。新 boolean 在线上用 `z.boolean()`，SQLite 0/1 在服务边界转换；复用 primitives，不新增 `number | string`、任意 record 或 passthrough 掩盖未知形状。shared 不导入 client/server、不引入 `node:` 运行时 API；纯同构函数可就近放在领域目录。现有 `isomorphic-dompurify` 是已存在的运行时依赖，不应误写为全包只依赖 Zod。

## 既有协议不能顺手美化

- **历史兼容**：`shared/src/weather/weather.schema.ts` 的 `weatherQuerySchema` 保留字符串 lat/lng，lang 默认 `de`。`weather.schema.spec.ts` 明确验证字符串和默认值。改为数字会改变现有接口；特定错误文案由 controller 保持，而非任意改用 Zod 默认错误。
- **历史兼容**：`shared/src/trip/trip.schema.ts` 的 `tripSchema.is_archived` 为 number，更新请求接受 boolean/number。这是旧协议，不是新增 boolean 的模板；文件中的旧 routes/services 注释不是当前源码入口。
- **当前实现**：`shared/src/collection/collection.schema.ts` 的 `collectionLinkSchema` 限 http/https；新请求对图片/网站有约束，但输出 `collectionPlaceSchema` 仍容许旧行字符串。输入安全与旧数据读取是两种责任，不要机械共用一个收紧后的 Schema。
- **历史兼容**：`shared/src/realtime/events.schema.ts` 注册表中的实体仍有 `z.unknown()`、ID 有 union；修改事件必须核对实际发射端，不得只把类型断言变窄就宣称安全。

## 不隐含单位与日期语义

`shared/src/budget/budget.schema.ts` 的 `budgetItemPayerSchema.amount` 使用费用自己的币种；不要推断项目所有金额都是整数分。新增字段应明确币种与转换点，并校验请求响应一致。

`shared/src/datetime/datetime-normalize.ts` 的 `splitLocalDateTime`、`normalizeLocalDateTime` 处理导入文档的 AM/PM 当地时间；不可用固定 slice 丢掉 PM，也不可统一转 UTC 改变预订含义。无法识别的输入有保持原值的兼容行为，收紧前读对应 `datetime-normalize.spec.ts`。

## 构建是公共边界的一部分

`shared/tsdown.config.ts` 构建 root、i18n metadata、每个 locale 三类入口，输出 ESM/CJS 与声明，Zod 不打包。`shared/package.json` 的 exports 区分 import/require，公开 `@trek/shared`、`@trek/shared/i18n` 和 locale 子路径。

添加公开符号要检查 root barrel；添加语言要检查 locale barrel 和构建 entry。消费者导入编译产物，不能引用仓库相对 src 路径来规避尚未构建的缺失导出。`client/vite.config.js` 将 shared 排除在 optimizeDeps 之外并监听 shared/dist，避免开发预打包缓存旧 Schema。

## 验证

在根目录运行 `npm run test --workspace=shared -- src/weather/weather.schema.spec.ts`（或所改领域 spec）、`npm run typecheck --workspace=shared`。共享代码改动后 `npm run build --workspace=shared`，再执行 server/client typecheck。这些是产品开发门禁，纯文档更新不必构建。
