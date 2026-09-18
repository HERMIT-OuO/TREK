# SDK 与宿主一致性

适用范围：插件权限、RPC、事件、manifest、网络规则、UI 桥与 registry 检查。依据 `plugin-sdk/CLAUDE.md`，目标是防止本地检查放行、安装后拒绝的假通过。

## 当前实现：生成事实

`server/scripts/gen-plugin-facts.ts` 直接导入两类来源：

- `server/src/nest/plugins/protocol/envelope.ts`：权限、方法、Hook 授权映射。
- `server/src/plugin-event-sink.ts`：事件 snapshot 授权与实体 ID 映射。

输出 `plugin-sdk/src/generated/host-facts.ts` 与 `shared/src/plugin-permissions.ts`。生成器还检查 shared 英文权限文案和 SDK `PERMISSION_FAMILIES` 覆盖。

**新增代码约束**：改源后使用生成器，绝不手改输出。生成文件随仓库保存，独立 SDK 构建不依赖旁边有 TREK server。生成器有意保留宽 `Record<string, string>` 类型；随意收窄为 literal union 也可能破坏作者源码兼容。

## 当前实现：手工镜像及验证范围

| SDK 位置 | 另一端 / 检查 |
| --- | --- |
| `plugin-sdk/src/egress-policy.ts` 的纯 policy helper | `server/src/nest/plugins/runtime/egress-policy.ts`；permissions-parity 比较指定函数 |
| `plugin-sdk/src/manifest.ts` | `server/src/nest/plugins/install/manifest.ts`；部分字段有 parity，完整规则仍需手工核对 |
| `KNOWN_ADDONS` | `server/src/addons.ts`；不能误认为由 host-facts 生成 |
| SDK session 常量、`plugin-sdk/src/cli/dev.ts` 预览桥 | `client/src/components/Plugins/PluginFrame.tsx`；检查限额和错误码 |
| `plugin-sdk/src/cli/checks/` | 独立 TREK-Plugins registry 的校验脚本；checks-parity 有条件运行 |
| `plugin-sdk/src/lucide-icon-names.ts` | `plugin-sdk/scripts/gen-lucide-icon-names.mjs` 从仓库根 lucide-react 生成快照 |

`plugin-sdk/test/permissions-parity.test.ts` 的 egress 比较会归一化空白，并只比较列出的函数；不是完整文件 byte-for-byte 保证。SDK 另有可恢复的 `installEgressGuard`，不复制宿主全部加固。不要根据旧源码注释引用不存在的 egress-parity 测试名。

## 当前实现：mock 保真有明确边界

`plugin-sdk/src/mock-host.ts` 的 `createMockHost` 提供 grants、actingUserId、成员/应用权限、Addon 开关与 fixture；driver 对 hooks/events/jobs 也做授权判断。

- mock db 是记录器与预设结果，不是 SQL 执行器；不能用它证明 SQLite 事务语义。
- `plugin-sdk/test/sdk.test.ts` 覆盖成员权限和 session scope/容量；`test/mock-limits.test.ts` 覆盖 daily budget 与 metadata 限额及校验顺序。
- mock 的 daily budget 是实例生命周期内计数，不在 UTC 零点自动重置；错误文字提到 midnight 不等于 mock 有计时器。
- `plugin-sdk/src/egress-policy.ts` 明确 dev 追求行为一致而非恶意代码隔离：可恢复 wrapper 不等价于生产 child 防护。

## skip 不是通过的证据

- `permissions-parity.test.ts` 缺宿主 envelope/egress 时跳过服务端组；缺客户端 PluginFrame 时跳过 frame 组。
- `plugin-sdk/test/checks-parity.test.ts` 需要 registry checkout；可用 `TREK_PLUGINS_REPO` 指向它。skip 条件具体是缺少 `scripts/validate-entry.mjs`；若它存在但 `check-readme.mjs` 缺失，相关测试会读取失败而非自动 skip。即使位于 TREK 仓库也不保证有完整 registry。
- registry parity 执行真实校验脚本时使用 `SKIP_NETWORK=1`；它证明离线子集，不证明远端 release、签名来源和下载状态全部通过。
- 不为消除 skip 自动 clone、上传代码或执行发布；报告缺失条件，维护者按任务需求提供环境。

## 修改顺序与检查

1. 确定来源与所有副本，说明当前行为和兼容语义。
2. 更新权威来源，再生成或同步副本；公开 API、mock、dev preview 和宿主都要考虑。
3. 比较完整集合，不只检查“包含旧条目”；权限选择器还需分类及解释文案。
4. 核对拒绝和错误顺序；不能通过放宽 mock、降低校验 severity 掩盖不一致。

在 TREK 根运行 `npm run check:plugin-facts --workspace=@trek/server` 为只读漂移检查；`npm run gen:plugin-facts --workspace=@trek/server` 会写文件，只在确实修改协议时使用。

在 `plugin-sdk/` 运行 `npm test -- test/permissions-parity.test.ts test/checks-parity.test.ts test/permissions.test.ts test/mock-limits.test.ts`，逐项记录 executed/skipped。宿主集成要求参见[服务端边界](../../server/backend/integrations-and-plugins.md)。
