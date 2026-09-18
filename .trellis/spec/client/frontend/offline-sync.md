# 离线缓存与同步

适用范围：已实现离线的行程数据及新领域的离线设计；不宣称管理员操作、所有 Addon 或文件操作均可离线。

## 当前实现：缓存边界

- `client/vite.config.js` 对业务 API 的 runtimeCaching 使用 `NetworkOnly`，不在 Service Worker 按 URL 缓存登录用户响应。其他排除路径也不能被描述成业务离线缓存。
- 业务离线数据由 Repo + Dexie 管理；SW 缓存应用壳、资源和匹配规则的地图资源，是不同层。
- `client/src/db/offlineDb.ts` 使用 `trek-offline-u<userId>` 用户库和匿名库；`offlineDb` Proxy 保持导入绑定、转向当前连接。
- 数据库名、表名和索引是设备契约；修改走追加版本迁移，不任意重命名。`version(3)` 的 blobCache 升级补 tripId/bytes 是实例。
- `client/src/sync/authGate.ts` 提供登录开关，让退出先停同步再切库。新增后台任务也需考虑在途请求返回时的账号切换，不仅检查启动时身份。

## 当前实现：读降级

`client/src/repo/withOfflineFallback.ts` 的 `onlineThenCache` 在有效离线或 Axios 无 response 的网络失败时读取缓存；HTTP 错误和非 Axios 错误仍抛出。它不用于写入。

网络判断统一用 `client/src/sync/networkMode.ts` 的 `isEffectivelyOffline()`：真实断网或用户强制离线。探活失败不是所有 Repo 读缓存的硬开关，避免一个探活请求失败使所有读取退到空缓存。

## 当前实现：离线写入与负 ID

以 `client/src/repo/placeRepo.ts` 为例：

1. CREATE 用 `nextTempId()` 生成会话内单调、不碰撞的负 ID，并写 Dexie 乐观行。
2. `generateUUID()` 生成队列 ID，`enqueue` 记录 method/url/body/resource/tempId；createdAt 也保持确定顺序。
3. UPDATE 合并缓存仍保留 `trip_id`；无缓存时也需这个字段，否则离线查询和清理都找不到该行。
4. 修改/删除尚未同步的负 ID 使用 URL `{id}` + `tempEntityId`，不能直接向服务器发负 ID。
5. 已支持版本控制的更新记录 `baseUpdatedAt`，来自编辑时的 `updated_at`；不是任意客户端时间。

`client/src/repo/placeRepo.offline.test.ts` 的 FE-REPO-PLACE-001～007 覆盖负 ID、队列形状、版本令牌及依赖创建的修改/删除。

## 当前实现：回放与幂等

`client/src/sync/mutationQueue.ts` 的 `flush` 入口检查认证/有效联网，循环每步复查认证；模块内 `_flushing` 防止同一运行实例重入，并不是跨标签页的数据库锁。它按 pending 的 createdAt 顺序回放，恢复超时遗留 syncing 行。`nextTempId` 的单调保证也限于当前会话，不能据此宣称跨标签页、重载或时钟回拨均绝不碰撞。

- `X-Idempotency-Key` 使用稳定的 mutation UUID；重试不得重新生成。`api/client.ts` 在线请求拦截器仅在未提供该头时补一个键。
- 服务端 CREATE 返回真实 ID 后删除临时行、保存规范实体，同时持久化改写依赖队列 URL/entityId，跨刷新仍有效。
- 新 `updated_at` 同时推进内存和队列兄弟更新的令牌，防止同一用户连续编辑互相冲突。
- 当前响应回填取响应对象首个值作为实体，且 `getTable` 只列出支持资源；新资源必须核对响应包裹及映射，不能假定任何响应都可回填。
- 网络/5xx 及 401/408/425/429 恢复 pending 并中止本轮，等待重认证或下次触发；其他终止性 4xx 标 failed，乐观 CREATE 的幽灵行被移除。`pending()`/`pendingCount()` 包含 pending 和 syncing，但不计 failed/conflict；回放实际只选 pending。

## 当前实现：冲突不是一般失败

`X-Base-Updated-At` 用于服务端乐观并发控制。非 DELETE 的 409 分三种策略：

| 策略 | 行为 |
| --- | --- |
| `ask` | 保留 body 和 `conflictServer`，停在 conflict 等待用户决定 |
| `mine` / `resolveKeepMine` | 清 base token 后重排队，无条件覆盖；不是自动合并 |
| `server` / `resolveKeepServer` | 本地采用服务端实体并移除本次写入 |

当前 DELETE 路径按 delete-wins、不参与同样的 CAS 冲突处理；不要把更新规则概括成所有动词相同。

## 新增代码约束与常见错误

依据 `client/CLAUDE.md`，新增离线领域需完整设计乐观写、入队、回放、失败反馈和恢复；在线专用须明确说明。不要把现有回填的类型断言、非原子步骤或吞掉缓存写错当作推荐设计。

WS 更新须与 Slice 一致并写透 Dexie，详见[状态规范](state-management.md)。同步不是只刷新页面：`client/src/sync/tripSyncManager.ts` 管理 bundle、文件缓存、过期/固定行程；缓存容量和预下载是否完成要单独检查。

## 验证

在 `client/` 运行 `npm test -- src/repo/placeRepo.offline.test.ts tests/unit/sync/mutationQueue.test.ts tests/unit/sync/mutationQueue.conflict.test.ts`，改库再加 `npm test -- src/db/offlineDb.test.ts`。

冲突测试直接覆盖 ask、keep-mine、keep-server 和连续更新不自冲突。新行为还需覆盖账号切换、重载后依赖 ID、HTTP 错误不退缓存及重连 WS；不要用“浏览器离线按钮能打开页面”替代这些数据断言。
