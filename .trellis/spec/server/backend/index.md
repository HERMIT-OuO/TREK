# server/backend 开发规范

适用范围：`server/src/`、`server/tests/` 与服务端配置/脚本。Nest 是业务应用，Express 只是 HTTP 适配器及少量平台中间件；HTML/插件页面交付也是服务端边界，不另设虚构的 React frontend 层。

## Pre-Development Checklist

- 阅读[目录与装配](directory-structure.md)、[API 与鉴权](api-and-auth.md)及[错误处理](error-handling.md)。
- 涉及 SQL/迁移/金额读[数据库](database-guidelines.md)；新增环境变量或日志读[配置](configuration.md)和[日志](logging-guidelines.md)。
- 涉及 MCP、WS、外部请求、存储、定时任务或插件读[集成与插件](integrations-and-plugins.md)。
- 核对 `shared/` 契约及同领域 REST/MCP/客户端消费者，先构建 shared 再做产品类型检查。

## 主题

- [目录、DI 与启动装配](directory-structure.md)
- [SQLite、事务、迁移与金额](database-guidelines.md)
- [输入契约、鉴权、幂等](api-and-auth.md)
- [错误响应与失败可见性](error-handling.md)
- [日志与脱敏](logging-guidelines.md)
- [环境配置、密钥与运行时语义](configuration.md)
- [MCP、实时、存储、调度与插件隔离](integrations-and-plugins.md)
- [质量检查](quality-guidelines.md)

## Quality Check

按[质量指南](quality-guidelines.md)运行定向测试、源码和测试 typecheck、只读 lint；build 不是类型门禁。参考 `server/CLAUDE.md` 的新增代码约束，但具体协议以当前实现与测试为准，不能把迁移注释或旧文件路径当成架构。
