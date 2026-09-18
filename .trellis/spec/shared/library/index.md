# shared/library 开发规范

适用范围：`shared/src/`、`shared/scripts/`、shared 构建配置。包名 `@trek/shared`；它是同构契约库，不是 React 前端，也不拥有业务数据库。

## Pre-Development Checklist

- 先读[跨包架构](../../guides/architecture.md)，定位真实生产者/消费者。
- 改 Schema 或纯函数读[契约与导出](contracts.md)，核对相邻 spec 和根 barrel。
- 改语言或 HTML 读[i18n 与净化](i18n-and-sanitization.md)。
- 变更后消费者使用 `dist`：确认构建顺序，不通过直接导入 src 绕过 exports。

## 主题

- [契约、primitives、类型与构建导出](contracts.md)
- [语言注册表、翻译和 HTML 净化](i18n-and-sanitization.md)
- [质量与验证](quality-guidelines.md)

## Quality Check

执行[质量指南](quality-guidelines.md)中与改动匹配的 Schema/语言测试及 typecheck；产品代码改动需要重建 shared 再检查消费者。重要约束来自 `shared/CLAUDE.md`；正文明确区分历史线协议与新契约标准，不把迁移遗留的宽类型当成新代码模板。
