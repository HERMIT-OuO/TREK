# 目录、DI 与启动装配

## 代码放在哪里

| 位置 | 职责与证据 |
| --- | --- |
| `server/src/index.ts` | dotenv 后先校验配置，再启动 `buildApp()`、监听同一 HTTP server；关闭流程协调 |
| `server/src/bootstrap.ts` | 生产与测试共享的 `buildApp()`；Express 平台中间件、解析器、Nest init 和启动检查 |
| `server/src/nest/app.module.ts` | 领域模块与全局 guards/filter/interceptors/pipe 装配 |
| `server/src/nest/collections/` | controller、DTO、service、module；带输入验证和领域访问控制的写入示例 |
| `server/src/nest/database/database.service.ts` | 通过 `DATABASE_CONNECTION` 注入同一 SQLite 连接；同步事务与查询接口 |
| `server/src/db/` | schema、追加式 migrations、seeds；不属于 ORM 模型目录 |
| `server/src/nest-mcp/` | MCP 装饰器/注册机制；领域工具放相邻 `.mcp.ts` |
| `server/tests/` | unit、integration、e2e、websocket，复用 helpers/fixtures |

## 新模块推荐形状

**新增代码约束**（`server/CLAUDE.md`）：薄 controller 处理协议，injectable service 拥有业务逻辑；纯计算放 helpers，有 repository 的领域把 SQL 留在 repository。模块显式声明 provider/import/export 并注册到 AppModule；跨领域通过注入公开能力，不新建函数模块“服务层”，不从 Nest 业务代码新导入全局 db/ws 单例。

**当前例子**：`CollectionsController` 注入 `CollectionsService`、`RuntimeEnvService`、`StorageService`；`CollectionsService` 注入 DB、Permissions、Realtime、Notifications 和 Storage。它不是通过 controller 传全套全局依赖，也不让请求体直达 SQL。

**历史兼容**：`server/src/nest/weather/weather.service.ts` 的 `WeatherService` 包装 `weather.impl` 并管理 cache cleanup 生命周期；`DatabaseService` 的部分权限辅助函数仍委托全局函数以保持旧 mock 行为。可复用模块形状，但不要新增这类单例/impl 包装当作统一架构。旧 `services/` 说明不能作为新的 import 路径。

## 启动顺序是契约

`buildApp()` 在 `app.init()` 前挂载全局中间件、上传、MCP discovery middleware、静态资源、文档和命名 body-parser；Nest init 后的未命中请求由 filter 终结，不能在后面补 Express route。

- 不向 pre-init 壳添加普通业务接口：它会绕过 Nest 全局安全管线。
- `/mcp` 必须保留原始 stream，不能 `@Body()`；`jsonParser` / `urlencodedParser` 的名称是 ExpressAdapter 去重依据，不随意重命名。
- 普通 JSON 限额与 photo book 的专用较大限额是不同路径，不为一个大请求放开全局上限。
- `getHttpServer()` 返回 WS adapter 绑定的实例，不能由入口/测试另建 HTTP server。
- `validateBodyContracts`、`validateRouteGuards`、`validateManagedRoutes` 在 init 后检查实际路由；不能为了通过启动扩张历史 allow-list。

## 验证

新增模块用真实应用 harness 验证 DI、路由/DTO 门禁及状态码；单独 `new Controller(mock)` 无法覆盖 metadata 装配。根目录执行 `npm run typecheck --workspace=server`、`npm run typecheck:tests --workspace=server`，再运行所改领域 unit/e2e，见[质量指南](quality-guidelines.md)。
