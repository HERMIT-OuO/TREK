# 状态管理与远端事件

适用范围：客户端本地 UI 状态、Zustand 领域状态、Repository 缓存和 WS 更新。

## 当前实现：分工而不是多份真相

| 状态 | 归属与依据 |
| --- | --- |
| 页面表单、弹窗与请求阶段 | 页面/特性 Hook，见 `useInstanceSettings` |
| 多视图共享行程 | `client/src/store/tripStore.ts` 的 `TripStoreState` |
| 领域写入和乐观更新 | `client/src/store/slices/` |
| 持久化离线实体 | `client/src/db/offlineDb.ts`，通过 Repo/同步层访问 |
| 网络覆盖开关 | `client/src/sync/networkMode.ts`，React 绑定用 `useNetworkMode` |
| 翻译和外观偏好 | Settings Store + 翻译上下文/主题应用，不在每个组件复制 |

`tripStore.ts` 组合多个 Slice，`resetTrip` 清掉行程相关状态，保留全局 tags/categories。新增行程字段需同时考虑加载、重置、离线 hydration 和事件更新，避免切换行程后显示前一行程的数据。

## 新增代码约束：订阅与数据路径

- 使用 selector；多字段读取按需要用 `useShallow`，不要整 Store 订阅。
- effect 内命令式读取用 `getState()`，不为了调用 action 订阅整个状态。
- 组件 → 特性 Hook → Store/Slice → Repo → API 或 Dexie；不得从 Modal 绕层写入 API。
- 新领域默认补读缓存和离线写入；若在线专用，明确记录原因和离线 UI。
- 约束源于 `client/CLAUDE.md`；已有直接 API 调用、整 Store 订阅是债务。

## 当前实现：读取与写入不是同一种降级

`client/src/repo/withOfflineFallback.ts` 的 `onlineThenCache` 只用于读：有效离线直接查 Dexie，在线请求遇到无 response 的 Axios 网络错误才退缓存；真实 HTTP 403/404/500 继续抛出。

写入失败不能套用该 helper 返回旧缓存假装成功。`client/src/repo/placeRepo.ts` 离线写乐观实体并入队，在线写采用 REST 返回的规范实体。不是所有领域都具有这一能力；读缓存不等于可排队写。

`tripStore.loadTrip` 对预算、文件等部分读取保留非致命降级，这是当前页面加载策略，不是允许新写入吞错。`budgetSlice.test.ts` 也记录旧 load 吞错，与新代码错误反馈要求要分开理解。

## 当前实现：WS 必须同时更新内存和缓存

`client/src/store/slices/remoteEventHandler.ts` 以两张表组织事件：

- `STATE_APPLIERS` 更新 Zustand；新增字段须与本地 Slice reducer 一致。
- `DEXIE_WRITERS` 在状态更新之后写透缓存；Day 中嵌套的 assignments/notes 必须从更新后的状态重建。
- `putPackingItem` 不仅跳过他人的私有条目，还删除失去可见性的旧缓存，避免离线继续泄漏。
- 现有写透是非阻塞并吞掉 Dexie 错误；这是缓存可用性取舍，不应复制到业务提交路径。

所有共享注册事件必须在 Store、专用监听器或显式忽略清单中有唯一决定。`client/tests/unit/remoteEventHandler/registry-parity.test.ts` 校验互斥分类、无陈旧名称、trip 作用域，以及每个 Dexie writer 均有 state applier。

## 常见错误与验证

只补本地 action、不补 WS，会让协作者看到不同结果；只补内存、不补缓存，会在脱网重开时回退旧数据。新字段也要覆盖删除、切换行程和重新连接。

在 `client/` 运行 `npm test -- src/store/slices/budgetSlice.test.ts tests/unit/remoteEventHandler/registry-parity.test.ts`，再按变更加入具体领域的 Repo/Slice 测试。离线队列的 ID、失败与冲突规则见[离线同步](offline-sync.md)；不要用 registry 枚举测试代替字段值行为测试。
