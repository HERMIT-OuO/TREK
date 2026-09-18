# 客户端开发规范

适用范围：`client/`，即 `@trek/client` npm workspace。当前为 React 19、Vite、Zustand、Dexie 和 Tailwind 的浏览器应用，不是服务端 React 层。

## Pre-Development Checklist

- 先读 `client/CLAUDE.md` 与本次涉及的主题；页面改动再读 `client/src/pages/PATTERN.md`。
- 找到桌面页面与 `client/src/mobile/screens/` 下对应手机入口，确认逻辑共享点。
- 确认数据是否属于离线行程领域；不能由“有 API”推断“已支持离线写入”。
- 改请求、响应或事件先读 [shared 契约](../../shared/library/contracts.md)；改宿主桥接同时读 [SDK 一致性](../../plugin-sdk/sdk/host-parity.md)。
- 新增 UI 先确认语义 token、错误边界、加载与失败状态，不复制旧硬编码样式。
- 先确定定向测试和命令副作用，再编辑；安装与启动见[本地开发](../../guides/local-development.md)。

## 主题导航

| 主题 | 何时阅读 |
| --- | --- |
| [目录与入口](directory-structure.md) | 新增功能、定位页面和数据层 |
| [组件与页面](component-guidelines.md) | 页面装配、桌面/手机 UI、懒加载 |
| [业务 Hook](hook-guidelines.md) | 异步生命周期、表单、复用逻辑 |
| [状态管理](state-management.md) | Store/Slice/Repo 边界、远端事件 |
| [离线与同步](offline-sync.md) | Dexie、负 ID、回放、幂等、冲突 |
| [主题与地图](theme-and-maps.md) | 外观 token、GL 分包、地图预下载 |
| [类型与内容安全](type-safety.md) | 共享类型、运行时校验、HTML/Markdown |
| [质量检查](quality-guidelines.md) | lint、类型、定向测试、E2E 与构建 |

## 规则的证据等级

- **当前实现**描述源码确实执行的行为，不保证所有旧文件一致。
- **新增代码约束**来自 `client/CLAUDE.md`、页面模式文档及现有检查脚本。
- **历史兼容/债务**单独说明；`any`、直接 API 写入和整 Store 订阅不是新代码范例。
- 源码引用均为仓库根相对路径；规范间链接相对当前文档解析。

## 最短阅读路径

- 只改布局：组件 → 主题 → 定向页面测试。
- 改业务操作：Hook → 状态 → 离线 → 类型与质量。
- 改 WS：共享事件契约 → 本地 Slice → `remoteEventHandler` → registry parity 测试。
- 改插件 UI：组件安全 → `PluginFrame` → SDK/宿主一致性，不将插件 HTML 内联到页面。

## Quality Check

在 `client/` 运行适用的 `npm run lint:check`、`npm run lint:pages`、`npm run typecheck` 和定向 `npm test -- <测试路径>`。先构建变动的 shared，再检查客户端；完整顺序见质量指南。

页面检查针对默认导出及脚本识别的桌面/手机容器，不禁止同文件子组件拥有局部状态。主题扫描默认只报告，成功退出不代表零违规。文档任务不需要启动服务或执行产品构建；记录实际执行项，不把规范中的示例命令当作已通过结果。
