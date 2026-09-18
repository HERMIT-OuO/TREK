# 组件与页面

适用范围：`client/src/pages/`、`components/` 与 `mobile/` 的渲染和交互装配。

## 新增代码约束：Page 是装配容器

`client/src/pages/PATTERN.md` 要求页面读取 `use<Page>()` 返回值并渲染 JSX。状态、effect、memo、ref、handler 和数据加载属于业务 Hook；允许 `useTranslation()` 等上下文读取及 guard 后的展示派生。

`client/scripts/check-page-pattern.mjs` 实际扫描页面默认导出，以及顶层 `*PageDesktop`/`*PageMobile` 容器。它不是全文件禁 Hook：同文件的展示子组件、辅助 Hook 不在这一禁令内。

- `client/src/pages/TripPlannerPage.tsx` 的 `ListsContainer` 自己管理子标签状态，是子组件边界的实际例子。
- 脚本的 `KNOWN_ESCAPES` 保留 TripPlanner 的特定旧声明；不要靠新增白名单绕过模式。
- 单纯提取 Hook 时按 PATTERN 保持 JSX 不变，不把架构整理和 UI 重设计混在一起。

## 桌面与手机

**当前实现**：`client/src/App.tsx` 根据 `useIsPhone` 分流懒加载页面，`MobileShell` 包住受保护页面；手机有独立屏幕，不只是桌面 CSS 缩小。

**新增代码约束**：两套壳保留各自布局，共享状态转换、校验、API 操作和派生算法。

- `client/src/components/Admin/useInstanceSettings.ts` 是两种设置壳共用的逻辑入口，处理脏表单先保存、危险动作确认、动作结果。
- `client/src/pages/admin/AdminSettingsTab.tsx` 与 `client/src/mobile/screens/admin/MAdminSettingsSection.tsx` 都接收 `useAdmin` 结果，但仍有直接请求和重复 handler；这是迁移债务，不是推荐复制模式。
- Props 显式描述组件需要的数据/回调；不要将大 Hook 对象不加选择地传遍整个树。

## 当前实现：懒加载与错误隔离

- `App.tsx` 使用 `lazyWithRetry`，登录页保持静态入口；新增路由跟随现有加载边界。
- `TripPlannerPage.tsx` 的 `LazyPanel` 将 `ErrorBoundary` 放在 `Suspense` 外面。Suspense 处理等待，拒绝的加载 Promise 由错误边界处理。
- 每个面板独立隔离，避免一个标签加载失败替换整个 Planner。
- `App.tsx` 的路由边界放在壳内并以 pathname 为 key，使失败后仍可导航离开。

## 插件与不可信内容

- 第三方页面只通过 `client/src/components/Plugins/PluginFrame.tsx` 交付，iframe 使用 `sandbox="allow-scripts allow-forms"` 与 `referrerPolicy="no-referrer"`。
- 不添加 `allow-same-origin` 来绕开桥接，不把插件 markup 注入宿主 DOM。
- iframe 按插件 ID 重建，导航后的 frame 不继续接收桥接；主题及上下文通过现有消息协议传递。
- 普通 Markdown、翻译 HTML 和地图弹层遵守[类型与内容安全](type-safety.md)，不能因为不在 React JSX 中就跳过净化。

## 样式与交互

新增样式使用[主题规范](theme-and-maps.md)的语义 token。复用 `client/src/components/shared/` 的 Modal、ConfirmDialog、Toast、ErrorBoundary，不自己创建 window 事件总线。保留明确的加载、失败和禁用状态，避免把请求失败伪装成成功关闭。

## 验证

在 `client/` 运行 `npm run lint:pages`、`npm run typecheck`，以及 `npm test -- src/pages/AtlasPage.wiring.test.tsx` 或实际改动页面的测试。

Wiring 测试要 mock Hook 验证 JSX 连接和交互；不能据此推断真实数据加载、地图生命周期已通过。桌面和手机至少各检查一次；共享行为单独测试，参见[Hook 规范](hook-guidelines.md)。
