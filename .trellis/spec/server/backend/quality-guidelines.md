# 服务端质量检查

## 命令与副作用

依据 `server/package.json`，从仓库根运行；先有 shared/dist。

```bash
npm run typecheck --workspace=server
npm run typecheck:tests --workspace=server
npm run lint:check --workspace=server
npm run test --workspace=server -- tests/unit/nest/trip-access.guard.test.ts
npm run test --workspace=server -- tests/unit/db/migration-version-atomicity.test.ts
```

- `typecheck` 检查源码，`typecheck:tests` 检查测试；Vitest 通过不说明 mock 类型通过。
- `lint` 有 `--fix`，用 lint:check 做不自动修复的验证。format 会写文件；format:check 不写，但脚本当前 glob 是 `src/**/*.ts` 与 `test/**/*.ts`，不能声称覆盖实际 `tests/` 全部文件。
- `build` 捕获 tsc 错误继续产出，不是类型门禁。
- `test:unit/test:integration/test:ws/test:e2e` 是 Vitest 分组；根 test:e2e 也是这一服务端套件。浏览器 Playwright 在 client。
- `test:coverage` 写 coverage。`check:plugin-facts` 只比较生成事实，`gen:plugin-facts` 会改 SDK/shared 生成文件；协议变化时应审查生成差异，不在普通验证中自动生成。

## 测试层级与代表证据

| 变更 | 首要证据/测试 | 还需验证 |
| --- | --- | --- |
| 授权 | `tests/unit/nest/trip-access.guard.test.ts` | 对应真实 e2e 的匿名、成员、陌生人、拒绝写入 |
| 幂等 | `tests/unit/nest/idempotency.interceptor.test.ts` | 重叠请求、实际 JSON 捕获、失败重试 |
| 迁移 | `tests/unit/db/migration-version-atomicity.test.ts` | 升级后数据与 schema_version 同步提交 |
| 配置 | `tests/unit/app-config/validate.test.ts` | 空白默认、错误聚合、运行时读取 |
| 日志 | `tests/unit/middleware/globalMiddleware.redaction.test.ts` | 新秘密字段及失败消息 |
| 插件页面 | `tests/unit/plugins/plugin-frame.test.ts` | 路径越界、CSP、真实 frame 加载 |
| MCP | `tests/unit/mcp/tools-days.test.ts` | 实际数据库和 REST/WS 语义一致 |

表内路径相对 `server/`。mock 测试只覆盖被测试层，权限、装配和持久化仍需真实 harness。

## 测试环境不能顺手改坏

`server/vitest.config.ts` 使用 SWC decorator metadata、forks 和 tests setup；替换为默认转译可能使 DI 失效。覆盖率使用 Istanbul，按领域配置阈值；具体数字以配置为准，不泛称所有领域达到统一 80%。已有约定要求阈值只升不降，不能为了交付删测试或降低 gate。

`server/src/db/database.ts` 在测试模式使用内存 DB；测试自建 DB 需要关闭，应用 harness 需要 cleanup，环境变更需要恢复，防止定时器、端口、插件子进程泄漏。文档任务不安装、不启动、不跑产品门禁；只检查文档路径和命令真实性。
