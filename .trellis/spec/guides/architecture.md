# 架构与数据流

## 包职责与依赖

**当前实现**（依据根 `package.json` 及各包 `package.json`）：

| 包 | 运行职责 | 依赖边界 |
| --- | --- | --- |
| client | React/Vite PWA；桌面与手机视图 | 通过 `@trek/shared` 使用契约；不导入服务端实现 |
| server | Nest 应用，Express HTTP 适配器，SQLite、WS、MCP、插件宿主 | 使用 `@trek/shared`；数据库、密钥、文件与网络权限留在服务端 |
| shared | 同构契约及辅助函数、翻译 | 不反向导入 client/server；不拥有页面或业务数据库 |
| plugin-sdk | 插件作者使用的 SDK、mock host 与 CLI | 独立安装/构建/发布包，不是根 npm workspace |

根 `dev` 先构建 shared，再并行运行 shared watch、server、client。根 `test` 仅覆盖三个 workspace；根 `test:cov` **额外显式执行** plugin-sdk 的 coverage，不能把“SDK 非 workspace”误解成“所有根脚本都忽略 SDK”。

## 一个业务变更经过哪里

新增客户端业务遵循 `组件 → 业务 Hook → Store/slice → Repository → API 或 Dexie`。这是 `client/CLAUDE.md` 对新代码的要求，不是所有旧页面都已离线化的事实。Page 默认导出的装配约束见 `client/src/pages/PATTERN.md`；领域的同步细节见[离线与实时](../client/frontend/offline-sync.md)。

后端由 `server/src/index.ts` 的 `bootstrap()` 调用 `server/src/bootstrap.ts` 的 `buildApp()`，使用同一 HTTP server 提供 REST 与 WS；不要另建 HTTP server 造成 WS 挂在未监听的实例上。`server/src/nest/app.module.ts` 的 `AppModule` 统一装配领域模块、全局 guard、filter、interceptor 和 Zod pipe。旧注释仍提 strangler/Express 迁移，但当前不是两套并行业务服务。

| 入口 | 责任 | 不能替代的部分 |
| --- | --- | --- |
| REST controller | HTTP 输入、DTO、身份、响应 | 不能把所有业务堆在 controller |
| MCP tool | 工具输入与访问策略；调用领域能力 | 不能绕过领域权限，也不能遗漏 REST 会发的实时变更 |
| WS | 广播变化、更新其他终端 | 不是数据库真相，也不保证断线期间消息重放 |
| Repository/Dexie | 用户隔离的业务离线缓存和乐观变更 | 不等于 Service Worker 缓存 API |

事件名字/负载的共同入口是 `shared/src/realtime/events.schema.ts` 的 `TREK_WS_EVENTS`。遗留事件存在开放实体与混合 ID 类型，只能逐项收紧，不能只改消费端断言。

## Addon 不等于 Plugin

`server/src/db/seeds.ts` 的 `seedAddons()` 注册内置功能开关；`AppModule` 中 `AddonsModule` 与 `PluginsModule` 是不同入口。Addon 是内置功能是否可用的配置。Plugin 是第三方扩展，涉及安装清单、授权、隔离执行、RPC 与页面交付；规则见[宿主集成](../server/backend/integrations-and-plugins.md)及 [SDK 协议一致性](../plugin-sdk/sdk/host-parity.md)。不能因为内置功能可直接读 DB，就把相同权限交给插件。

## 验证与常见误判

- 改共享字段：验证 shared 的 Schema 测试、构建导出及两端类型；不要靠前端本地 interface 掩盖协议漂移。
- 改写入：同时验证权限、事务、幂等、缓存与 WS；只看 HTTP 200 不够。
- 改插件 API：同时检查宿主与 SDK；SDK mock 通过不等于真实隔离安全通过。
- 开发命令的前提及副作用见[本地开发](local-development.md)。
