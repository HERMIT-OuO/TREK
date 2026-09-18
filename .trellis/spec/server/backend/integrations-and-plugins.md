# 集成与插件边界

## MCP 与 REST：相同业务，不同适配器

`server/src/nest/days/days.mcp.ts` 的 `DaysMcp` 展示本地模式：`@Tool` 声明 inputSchema、annotations、access；调用 DaysService，并做 demo、trip access、`day_edit` 检查；成功后发与 REST 一致的 day 事件。update 使用实际传入字段的 rest spread，不能把未提供 notes 变成显式 undefined 导致清空。

**新增代码约束**：同一次业务修改联查 REST 与 MCP 的权限、默认值、写入结果、实时 payload；复用 shared 字段 Schema，不在 MCP 内重写 SQL。工具名、scope 与 destructive/idempotent 提示也是公共协议。不是所有传输验证完全相同：当前 update_day 的 title cap 比 REST 更窄，属于明确的适配器约束；“一致”指业务语义，不是要求 HTTP 和 JSON-RPC 响应字节相同。

证据：`server/src/nest/days/days.controller.ts` 使用 `JwtAuthGuard → TripAccessGuard + @RequirePermission('day_edit')`；`DaysMcp` 则先检查 trip 可见性，再调用 `server/src/nest/mcp-shared/mcp-tool-guards.service.ts` 的 `hasTripPermission`。两者最终复用 `PermissionsService.checkPermission` 的相同 action、owner 与 shared 语义；工具 access scope 不能替代资源授权。`safeBroadcast` 为 MCP 消息补 `_source: 'mcp'`，私有数据应传 `onlyUserIds` 限定接收者，省略表示整间 trip room，并非自动识别行权限。

`server/tests/unit/mcp/tools-days.test.ts` 用真实 MCP harness 与内存数据库覆盖 title/notes 的独立更新和读回、跨 trip/非成员拒绝及 day:updated 广播；其中广播仍是 mock，不是实际 WS 连通测试。`/mcp` 原始 stream 要求见[装配指南](./directory-structure.md)。

## 实时事件

注入 `server/src/nest/realtime/realtime.service.ts` 的 `RealtimeService`；trip broadcast 带 tripId，user broadcast 负载自带 type。`X-Socket-Id` 对应 excludeSid，私有行可用 onlyUserId 缩小范围。名字/负载类型来自 shared 注册表，但当前 facade 是运行时透传，不能声称每条广播均执行 Zod parse。

不要捕获单例函数快照或让不同入口自行命名事件；保留可选参数的原有调用形状。生产 transport 已在 Nest realtime 域，facade 注释中“未来迁移”是历史文字。客户端同时处理内存和缓存，见[离线同步](../../client/frontend/offline-sync.md)。

## 存储、外部 URL 与调度

- **存储**：`StorageService` 是 `(category, name)` 门面，每次解析 registry；调用者不持有 driver/S3 客户端、不拼磁盘 uploads 路径。用 sendToResponse/withLocalFile 处理本地与远端差异。`server/tests/unit/nest/storage/storage-keys.test.ts` 固定 traversal、反斜杠、空段、隐藏段、控制字符等拒绝行为。
- **出站请求**：新 fetch 要有超时、大小上限及结果验证；普通用户 URL 使用 `server/src/utils/ssrfGuard.ts` 的 `safeFetch` / `safeFetchFollow`，逐跳 `checkSsrf`、DNS pin 并处理跨 origin 凭据，不先验一次后裸 fetch。ALLOW_INTERNAL_NETWORK 不放开这条路径的硬禁止地址。**受限例外**：同文件 `safeFetchAdminConfigured` / `safeFetchLlm` 为管理员配置的服务允许 LAN/回环，仍逐跳拒绝 metadata/link-local；不能拿这条宽路径处理普通用户提交的任意 URL。
- **调度**：领域 `*.job.ts` 通过 `CronRegistrarService` 注册；`server/src/nest/scheduling/cron-registrar.service.ts` 在 test 环境拒绝调度，关机逐项清理。不要直接新增 `@Cron` 绕过测试 gate。

## 插件宿主隔离

**当前实现**：`server/src/nest/plugins/supervisor/plugin-supervisor.ts` 的 spawn 用 child process + IPC、环境白名单；编译 JS 模式附加 permission 参数。`runtime/` 与 `supervisor/` 有意不是 Nest 业务 provider，不要为“统一 DI”消除进程边界，也不要把开发运行模式当成生产隔离等价证明。

`server/src/nest/plugins/host/rpc-host.ts` 的 `PluginRpcHost` 让 registry 仅绑定授权 method；未知方法和未授权已知方法分别返回 UNKNOWN_METHOD/PERMISSION_DENIED。RPC 的 acting user 由宿主解析，不信任插件自报 userId；具体资源还需要领域授权，拥有能力不等于可访问全站数据。参数先验证，再调用服务。

协议/权限生成源、手工镜像与 SDK 的联动见[host parity](../../plugin-sdk/sdk/host-parity.md)。安装 manifest、宿主版本范围、权限授权与运行时隔离是不同关卡，不能只跑 SDK mock 就宣称已验证宿主安全。

## 服务端 HTML 与插件页面

`server/src/nest/plugins/plugin-frame.controller.ts` 的 `PluginFrameController` 只给启用且 active 的插件交付 client 文件；路径同时做 lexical containment 和 realpath containment，防止符号链接越界。响应 CSP 限定插件资源与授权出站 host，使用 opaque sandbox，**不加 allow-same-origin**。CORP cross-origin 是为 opaque frame 加载自己的脚本，不是放弃隔离。

`server/tests/unit/plugins/plugin-frame.test.ts` 验证 sandbox、CSP、CORP、root-relative sendFile 和 operatorEgress host。服务端拼 HTML/headers 同样要转义和校验不可信插值；不能将插件 HTML 放进主站 DOM，也不能为修复页面加载把 CSP 改成全通配。

## 验证

根目录按主题选择 `npm run test --workspace=server -- tests/unit/mcp/tools-days.test.ts`、`tests/unit/plugins/plugin-frame.test.ts`、`tests/unit/nest/storage/storage-keys.test.ts`；WS 改动另跑 `npm run test:ws --workspace=server`。插件 API 同时检查 SDK parity，真实隔离须使用对应宿主集成测试并记录平台前提。
