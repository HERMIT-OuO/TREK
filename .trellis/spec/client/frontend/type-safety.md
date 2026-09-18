# 类型与内容安全

适用范围：客户端 API、WS、组件 Props、共享模型与不可信内容渲染。

## 当前实现：共享契约为主要来源

`client/src/types.ts` 重新导出 `@trek/shared` 的 Trip、Place、Day、BudgetItem 等实体，保留旧导入路径；其中仍有客户端 User/TodoItem/TripFile 等本地接口，不能声称它完全只是 barrel。

**新增代码约束**：跨端请求/响应优先在 shared Schema 定义并推导类型，客户端引用它，不凭 UI 使用习惯另造一份线协议。纯展示派生类型留在页面 Model 或组件附近。

- `client/src/api/client.ts` 导入共享请求类型、Zod Schema；新领域传输放领域 API 模块，不继续堆这个大文件。
- `client/src/pages/atlas/atlasModel.ts` 的可选 status 有旧服务器兼容语义，缺失时按 visited；不要用一次类型整理改变兼容行为。
- ID、日期、金额及 nullable 字段遵守共享 Schema；不凭“前端更好处理”转换存储单位或省略字段。

## 当前实现：类型声明不等于运行时拒绝

`api/client.ts` 的 `parseInDev` 只在开发模式 safeParse 并警告，失败和生产模式仍透传原数据；`checkInDev` 甚至保留调用方原有推导类型。

这是契约漂移提示，不是安全校验器，也不是生产 API 自动净化。真正需要拒绝不可信数据的入口应显式验证/缩窄，不能依赖一个类型断言或把 dev helper 描述成强校验。

Axios 统一使用 `/api`、`withCredentials: true`，请求拦截器补 socket ID 和幂等键；绕开该实例会丢掉认证和事件去重约定。

## 新增代码约束：边界不可用 any 掩盖

依据 `client/CLAUDE.md`：WS payload、地图 renderer Props、外部消息使用真实类型；不新增全局可变 window 状态或 window 事件总线。

`client/src/store/slices/remoteEventHandler.ts` 已按共享事件名注册，但内部仍有 Record/断言；`MapViewAuto`/`glLazy` 也有旧 any。这些是历史债务，不是新边界可以无检查传任意值的理由。

## 内容渲染安全

- 用户字符串不插进 HTML 模板，地图 popup/marker 也一样；优先 DOM + `textContent`，需要字符串时用 shared `escapeHtml`。
- `client/src/components/shared/markdownSanitize.tsx` 的 `sanitizedMarkdownPlugins` 先将 raw node 转文本，再 `rehype-sanitize`，保留用户输入的尖括号文本但不创建 HTML 元素。
- 使用同文件的 `sanitizedMarkdownComponents` 保留片段链接，外链带新标签页及 rel 保护；不在不可信 Markdown 附近引入 `rehype-raw`。
- `client/src/i18n/TranslationContext.tsx` 的 `tHtml` 先 escape 参数再净化模板；一般文本用 `t`，需要 markup 时走现有安全通道，不把 `t()` 结果直接当安全 HTML。
- 第三方插件走 `PluginFrame` iframe，不内联渲染插件 markup，详见[组件规范](component-guidelines.md)。

## i18n 与浏览器边界

翻译字符串属于 shared。`TranslationContext.tsx` 静态加载 en、为其他语言显式动态 import，保持语言分包；新增语言同时检查 shared 注册表和客户端 loader。

客户端模块会使用 document、navigator、IndexedDB，不能被 shared 反向导入而形成端侧依赖；客户端导入 shared 契约则是正常方向。`useNetworkMode` 的服务端快照参数只是 Hook 的兼容边界，不表示整个客户端支持 SSR。

## 验证

从根目录先 `npm run build --workspace=@trek/shared`，再在 `client/` 执行 `npm run typecheck` 与 `npm run lint:check`。shared 未变且产物有效时无需反复构建。

按改动补请求/响应、恶意 HTML、Markdown 尖括号保留和语言切换的行为测试；WS 事件补 `npm test -- tests/unit/remoteEventHandler/registry-parity.test.ts`。规则及构建顺序同时参见[shared 契约](../../shared/library/contracts.md)与[客户端质量检查](quality-guidelines.md)。
