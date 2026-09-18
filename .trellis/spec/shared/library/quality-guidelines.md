# shared 质量检查

## 按变更选择门禁

依据 `shared/package.json`；下列从仓库根运行，需已有兼容依赖。

```bash
npm run typecheck --workspace=shared
npm run test --workspace=shared -- src/weather/weather.schema.spec.ts
npm run test --workspace=shared -- src/sanitize/sanitize.spec.ts
npm run i18n:parity:strict --workspace=shared
npm run format:check --workspace=shared
# 只检查；不同于包 lint 脚本的 --fix
npm exec --workspace=shared -- eslint "src/**/*.ts"
```

示例测试文件按修改领域替换；全包测试是 `npm run test --workspace=shared`，coverage 会产生报告。新增 Schema 要覆盖有效输入、缺失、null、错误类型、默认值及兼容边界；不要只测试 `safeParse` 能运行。

## 新代码与历史例外

`shared/CLAUDE.md` 要求 strict TypeScript、无新增 any/压制注释。现有 JS 脚本导入处的 `@ts-expect-error` 或事件实体的 unknown 不构成扩大宽类型的理由。类型断言不能代替边界解析；Schema 改动必须说明真实请求/响应的兼容性。

`shared/src/weather/weather.schema.spec.ts` 是精确协议例子；`shared/src/sanitize/sanitize.spec.ts` 是攻击输入与允许输入并测的例子。i18n 普通测试与 strict parity 的区别见[i18n 指南](i18n-and-sanitization.md)。

## 构建和生成文件

- Schema/locale/导出修改后执行 `npm run build --workspace=shared`，再执行消费者 typecheck。build 清理并写 dist；watch 使用 `--no-clean`，不要在缺失导出时只看源码。
- `npm run lint --workspace=shared` 带 `--fix`，`format` 带 `--write`，会改文件。
- `shared/src/plugin-permissions.ts` 是生成文件；来源与只读 drift 检查见[宿主一致性](../../plugin-sdk/sdk/host-parity.md)。不要手工修复输出。
- 纯文档修改只核对路径、链接与命令，不要求跑上述产品命令；未执行必须如实说明。
