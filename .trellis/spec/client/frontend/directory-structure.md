# 客户端目录与入口

适用范围：新增或定位 `client/src/` 下的功能。目录表示职责，不要求为了统一命名移动现有文件。

## 当前实现：从入口向下定位

| 路径 | 职责与真实入口 |
| --- | --- |
| `client/src/App.tsx` | 路由、鉴权外壳、桌面/手机分支、懒加载和主题应用 |
| `client/src/pages/` | `*Page.tsx` 页面装配；`PATTERN.md` 定义拆分约定 |
| `client/src/pages/atlas/` | `useAtlas.ts` 业务 Hook、`atlasModel.ts` 非 React 模型 |
| `client/src/components/` | 按业务域组织，`shared/` 放跨域 UI |
| `client/src/mobile/` | `MobileShell`、手机屏幕、手机组件与局部 token |
| `client/src/hooks/` | 跨页面 Hook，如 `useNetworkMode`、`useTripWebSocket` |
| `client/src/store/` | Zustand；`tripStore.ts` 装配 `slices/` 中的行程操作 |
| `client/src/repo/` | 网络与 Dexie 读取选择，以及已支持领域的离线写入 |
| `client/src/api/` | Axios/WS 与领域传输接口，不负责 UI |
| `client/src/db/`、`client/src/sync/` | 缓存模型、写入队列、联网与同步生命周期 |
| `client/src/theme/` | 外观应用与方案元数据 |
| `client/src/i18n/` | React 翻译上下文；翻译内容属于 shared |
| `client/src/managed/index.tsx` | 托管安装的构建替换点，不是通用功能目录 |

## 新增代码约束：选择最小落点

1. 页面保留装配，状态、派生值、加载和 handler 放同目录 `use<Page>`。
2. 无 React 依赖的类型和算法放 Model 或已有工具模块；跨前后端契约放 shared。
3. 桌面和手机共用逻辑放一个 Hook/模块，不各自复制请求和表单规则。
4. 新 API 面放 `client/src/api/` 的领域模块；不要继续扩大 `api/client.ts`。
5. 行程数据沿 Hook → Store/Slice → Repo → API/Dexie 访问，不能从 Modal 直接写 API。
6. 这些约束来自 `client/CLAUDE.md`，不是“目前所有文件已遵守”的声明。

## 测试与资源的位置

- `client/vitest.config.ts` 同时收集 `tests/**/*.test.{ts,tsx}` 与 `src/**/*.test.{ts,tsx}`。
- 页面 wiring 可参考 `client/src/pages/AtlasPage.wiring.test.tsx`；Slice 参考 `client/src/store/slices/budgetSlice.test.ts`。
- Dexie 相关测试使用 fake-indexeddb，HTTP 行为通过 `client/tests/helpers/msw/server.ts` 隔离。
- 浏览器流程位于 `client/e2e/`，与 Vitest 分开；截图不是组件断言的替代品。
- `client/public/theme-boot.js` 是首屏前的外观快照回放，不属于一般组件入口。

## 常见错误

- 把 `mobile/` 当成另一套业务后端，造成桌面/手机校验和保存顺序漂移。
- 把 Atlas Model 的共享可变查找表当成所有 Model 都可存全局状态的许可。
- 因 `api/client.ts` 有许多旧领域接口而继续把新代码塞进去。
- 改 Dexie 数据库或表名如同普通内部重命名；这是用户设备上的持久化契约。

## 验证

新增页面后在 `client/` 执行 `npm run lint:pages`、`npm run typecheck` 和对应测试。只改目录前检查导入、路由分支和两个端的入口；涉及持久化先读[离线规范](offline-sync.md)，命令选择见[质量检查](quality-guidelines.md)。
