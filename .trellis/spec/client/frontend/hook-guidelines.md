# 业务 Hook

适用范围：页面逻辑 Hook、跨端共用状态机和浏览器订阅。依据 `client/CLAUDE.md` 与 `client/src/pages/PATTERN.md`。

## 职责与返回值

**新增代码约束**：`use<Page>()` 拥有状态、effect、ref、memo、handler、Store selector 与加载，返回页面装配所需的单个对象。页面负责渲染，纯规则进入不依赖 React 的 Model。

- `client/src/pages/atlas/useAtlas.ts` 展示地图实例、异步图层和页面状态如何离开 Page。
- `client/src/pages/atlas/atlasModel.ts` 的 `countryStatus`、`withCountryMarkedVisited` 可独立测试，兼容缺少 status 的旧数据。
- 同文件 `A2_TO_A3` 是历史共享可变查找表；不要推广为通用全局状态容器。
- 拆出 Hook 不等于完成分层：`useAtlas` 仍很大并有直接 API/旧类型；新逻辑按关注点拆分，不制造另一个巨型 Hook。

## 跨端只维护一条行为路径

`client/src/components/Admin/useInstanceSettings.ts` 的 `useInstanceSettings` 处理：

1. 打开时加载字段、脱敏值和可用动作。
2. `persist` 校验并保存；`settingsPatch` 避免未编辑的 secret mask 覆盖服务端密文。
3. 脏表单先保存，再执行动作；`danger` 动作等待壳上的确认。
4. `openSeq` 防止旧动作结果写入新打开的对话框，引用比较保留保存过程中产生的新编辑。

这是共享行为的例子，不表示其中每个异步分支都已完整防竞态。新增/修改加载分支仍要检查关闭、切换实体及卸载场景。

## 外部订阅与异步生命周期

**新增代码约束**：优先使用已有外部状态入口，不复制浏览器监听器和本地镜像。

- `client/src/hooks/useNetworkMode.ts` 用 `useSyncExternalStore` 订阅 `onNetworkModeChange`；业务代码读 `isEffectivelyOffline()`，不直接读 `navigator.onLine`。
- `client/src/i18n/TranslationContext.tsx` 的语言加载 effect 使用 cancelled 标志，并在语言变更时清理，避免旧 chunk 覆盖新语言。
- 新异步 `.then(setState)` 使用取消标志、序列检查或 AbortController；事件、地图实例、observer 必须有对应清理。
- 不惯例性关闭 exhaustive-deps；依赖遗漏会让金额、权限等派生值停留在旧输入。
- React 19 的 `useOptimistic` 等可按场景使用，但不能替代已有持久化队列、服务端幂等和错误回滚。

## 数据与错误边界

新行程业务由 Hook 调用 Store/Slice，经 Repo 访问数据，详见[状态管理](state-management.md)。明确在线专用场景，不假装所有 Hook 都能脱网写入。

新增乐观操作需要失败回滚和用户可见反馈；不能 `.catch(() => {})` 静默丢掉写入。独立请求可并发，不新增串行 N+1 加载循环。每个操作明确由哪一层展示错误，避免重复 toast。

## 定向测试

在 `client/` 运行：

```bash
npm test -- src/components/Admin/useInstanceSettings.test.ts
npm run typecheck
npm run lint:check
```

`useInstanceSettings.test.ts` 覆盖脏表单保存顺序、保存失败阻止动作、危险动作确认、进行中的编辑不被覆盖和旧动作结果隔离。新增 Hook 测试应直接驱动这些状态，而不是只 mock Hook 后渲染 Page；Page wiring 属于另一层测试。
