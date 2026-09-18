# TREK 跨包开发指南

适用范围：仓库根目录及 `client/`、`server/`、`shared/`、`plugin-sdk/` 之间的协作。这里说明真实边界，不代替包级规则。

## Pre-Development Checklist

- 先读任务需求，确认变更属于哪一包、是否改线协议、离线数据或插件权限。
- 阅读[架构与数据流](architecture.md)和[本地开发](local-development.md)。跨边界改动再读[跨层检查](cross-layer-thinking-guide.md)，新增工具函数前读[复用检查](code-reuse-thinking-guide.md)。
- 按代码落点进入下表索引，再读涉及的主题；不要把读完索引当成已读规范。

## 规范入口

| 入口 | 负责范围 |
| --- | --- |
| [client/frontend](../client/frontend/index.md) | React 页面、移动端、Store、离线与地图 |
| [server/backend](../server/backend/index.md) | Nest、鉴权、SQLite、实时、MCP、插件宿主 |
| [shared/library](../shared/library/index.md) | Zod 线协议、i18n、纯函数与构建导出 |
| [plugin-sdk/sdk](../plugin-sdk/sdk/index.md) | 独立 SDK/CLI、宿主镜像与生成数据 |

## 如何理解规则

- **当前实现**：可由指向的源码或测试核实，描述已有行为。
- **新增代码约束**：来自现有包级约定或检查脚本；旧代码未全部达标不代表可以继续复制。
- **历史兼容/债务**：保留线协议或迁移实现的例外，不是新领域的默认范式。

源码证据使用仓库根相对路径，并点名关键符号。源码注释中的旧 Express 路径、迁移阶段和过期计数不自动成为当前事实。冲突时先核对运行代码、配置及测试，再看局部约定，最后看根概览。

## Quality Check

按各包质量指南选择定向检查，跨包修改检查两端。核对请求、存储、响应、缓存和事件是否保留同一语义。文档变更检查链接、源码路径、命令及索引；无需为了纯文档修改启动产品或全量构建。检查结果应明确哪些执行了、哪些没有执行，不能用 build 成功代替类型或协议检查。
