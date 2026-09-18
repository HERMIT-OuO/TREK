# API 输入、鉴权与幂等

## 共享 Schema 必须进入实际管线

**当前推荐路径**：`server/src/nest/collections/collections.dto.ts` 包装 shared Schema：

```ts
export class CollectionCreateDto extends createZodDto(collectionCreateRequestSchema) {}
```

`CollectionsController.create` 的 `@Body() body: CollectionCreateDto` 让 `server/src/nest/common/zod-validation.pipe.ts` 的全局 pipe 根据 metatype 验证。只写一个 interface/类型别名不会留下运行时 DTO；pipe 对非 DTO 参数会原样放行。

`validateBodyContracts` 检查声明 `@Body()` / `@Body('field')` 的 POST/PUT/PATCH/**DELETE**，未用 createZodDto 且不在历史 allow-list 中则拒绝启动，过时 allow-list 也报错。它读取 decorator metadata，不扫描 handler 内的任意取值；新增端点不能扩大 allow-list 或改成 `@Req().body` 来绕过。Schema 合法也不代表所引用 ID 属于当前 trip。

保持 method、URL、字段缺省/null、响应状态、Cookie 与定制错误一致。静态子路由先于 `:id`；Nest POST 默认 201，仅在旧契约要求 200 的地方加 `@HttpCode(200)`，不要无差别改所有 POST。

## 默认拒绝与领域授权

`server/src/nest/app.module.ts` 顺序为 GlobalAuthGuard → MfaPolicyGuard → ManagedGuard。`server/src/nest/auth/global-auth.guard.ts`：

- 普通路由默认认证；`@Public(reason)`、`@OptionalAuth()` 显式例外。
- 声明自己的 guard chain 时，全局 guard 解析用户但让链自行决定拒绝；因此只加一个非认证 guard 不等于已鉴权。
- `server/src/nest/common/validate-route-guards.ts` 的 `collectRouteGuards` 枚举 public、optional 和声明 guard 的路由；`validateRouteGuards` 实际强制的是 `@Public` 与无认证 guard 链两份清单，含过时项检查，**不对 OptionalAuth 单独做清单门禁**。新增 optional 路由仍须人工审查匿名响应；清单中的 MCP、分享下载等入口可能在 handler 内验证另一类凭据，不能把 `@Public` 理解为无条件开放业务数据。

复用 `server/src/nest/auth/jwt-verify.ts` 的 `verifyJwtAndLoadUser`：验证 HS256、拒绝 purpose token 冒充会话、加载用户并核对 password_version。`decodeSessionClaims` 仅给**已认证**请求读元数据，绝不能作为认证。常规会话优先 httpOnly `trek_session` Cookie，Bearer 为其他客户端兼容通道。

`TripAccessGuard` 必须在认证后执行；不可见 trip 返回 404 `Trip not found`，可见但无操作权限返回 403 `No permission`。领域 service 仍需保留授权能力，MCP 不经过 HTTP guard。`CollectionsService.assertAccess/assertCanEdit` 同样区分不可见和只读成员。

测试证据：`server/tests/unit/nest/trip-access.guard.test.ts` 的 TRIPGUARD-002 固定陌生用户与不存在 trip 都为 404，避免枚举；不能为了“语义正确”统一改成 403。

## 上传的额外顺序约束

保留身份认证；不要把依赖 multipart 已解析数据的业务拒绝条件提前放到 guard。`server/CLAUDE.md` 指出 parser 之前拒绝上传可能让客户端看到 ECONNRESET 而非业务 403。按现有上传 controller 在 handler 中做相应业务检查，并处理已经落地的临时文件。文件 MIME、扩展名、大小、存储 key 和权限是不同检查，不得互相替代。

## 幂等键的真实保证

`server/src/nest/common/idempotency.interceptor.ts` 的 `IdempotencyInterceptor`：认证写入按 `(key, user, method, path)` 查缓存，同进程重叠请求等待在途结果；只捕获限额内的成功 JSON 响应。无用户/无键/非写方法放行，过长 key 返回 400。

**限制**：204、send/end、过大 body 或缓存存储失败不保证重放命中；业务写入与响应缓存不是一个数据库事务，不是跨进程或崩溃场景的 exactly-once 承诺。不要给同一个离线重试换 UUID，也不要为不同操作复用同一个键。

`server/tests/unit/nest/idempotency.interceptor.test.ts` 验证缓存重放跳过 handler、精确作用域及成功 JSON 捕获。新增写路径需验证自己的响应确实经过捕获点，并与[客户端回放](../../client/frontend/offline-sync.md)联查。

## 验证

根目录运行 `npm run test --workspace=server -- tests/unit/nest/trip-access.guard.test.ts tests/unit/nest/idempotency.interceptor.test.ts tests/unit/nest/validate-body-contracts.test.ts`，再用对应领域真实 e2e 检查未登录、无权限、非法输入、重复请求及静态路由命中。不要只用 controller mock 判定全局鉴权通过。
