# 规范目录与编写方案

## 1. 设计边界

此任务交付一个完整的 TREK 规范集及其导航，不是多个独立产品功能。沿用一个任务、一个负责者，按包依次编写，再统一核验跨包契约；暂不创建父子任务树。

允许修改范围：

- `.trellis/spec/`：规范正文与索引。
- `.trellis/config.yaml`：仅 packages/default_package 映射的必要修正。
- `.trellis/tasks/00-bootstrap-guidelines/`：规划、研究、上下文和验证记录。

不改产品代码、依赖、现有约定文件、Trellis 运行时、平台技能或提交历史。

## 2. 目标目录与内容职责

保留适用的文件名，移除与职责不符的模板；以下是目标主题清单，同一范围内可在编写时合并短小重复段落，但不可省略验收要求。

```text
.trellis/spec/
├── guides/
│   ├── index.md
│   ├── architecture.md
│   ├── local-development.md
│   ├── code-reuse-thinking-guide.md
│   └── cross-layer-thinking-guide.md
├── client/frontend/
│   ├── index.md
│   ├── directory-structure.md
│   ├── component-guidelines.md
│   ├── hook-guidelines.md
│   ├── state-management.md
│   ├── offline-sync.md
│   ├── theme-and-maps.md
│   ├── type-safety.md
│   └── quality-guidelines.md
├── server/backend/
│   ├── index.md
│   ├── directory-structure.md
│   ├── database-guidelines.md
│   ├── api-and-auth.md
│   ├── error-handling.md
│   ├── logging-guidelines.md
│   ├── configuration.md
│   ├── integrations-and-plugins.md
│   └── quality-guidelines.md
├── shared/library/
│   ├── index.md
│   ├── contracts.md
│   ├── i18n-and-sanitization.md
│   └── quality-guidelines.md
└── plugin-sdk/sdk/
    ├── index.md
    ├── api-and-cli.md
    ├── host-parity.md
    └── quality-guidelines.md
```

### guides

- architecture：包依赖、业务数据流、REST/MCP/WS 的角色、Addon 与 Plugin 区别；不堆砌全部业务功能清单。
- local-development：可移植的根目录命令、最小本地环境原则、首次登录、SQLite 数据位置、调试的 Source Map 前提、检查命令副作用。
- 两份 thinking guide 保留“先查复用、检查跨层影响”的有用内容，替换泛泛模板和无来源统计，加入 TREK 的契约/离线/插件案例。

### client/frontend

- Page 与 Hook、纯 Model 的职责，桌面/移动端逻辑复用、错误边界和懒加载。
- Store/Repo/API/Dexie 边界；离线写入的临时 ID、幂等键、回放与冲突，WebSocket 远端事件；明确非所有领域都实现同等离线能力。
- 主题 token、地图多渲染器及分包，契约类型、HTML/Markdown 安全。
- 测试区分页面 wiring、Hook/Store、离线缓存、UI 集成与 Playwright；质量命令按 package.json 实际脚本记录。

### server/backend

- Nest 领域模块、DI、装配顺序、接口/服务分工；明确现存 impl/单例为兼容例外，不鼓励新增。
- 输入 Schema/DTO、默认拒绝鉴权、Trip 权限和错误可见性、REST/MCP 一致性、幂等与实时广播。
- SQLite 参数绑定、事务、追加式迁移、测试隔离；错误封装、日志脱敏、配置解析。
- 以一个整合主题覆盖 MCP/实时/存储/调度/插件的边界与入口，必要时按阅读密度拆分；不把所有领域服务逐一写成规范。
- 服务端 HTML 输出、插件页面交付与不可信 URL 的安全要求归入后端相关主题，不能因删除 frontend 模板而遗漏。

### shared/library

共享层不拥有 React 页面和业务数据库。涵盖 Schema 与推导类型、真实线协议兼容、通用 primitives、构建 exports、语言注册表与翻译一致性、净化函数及无端侧依赖约束。

### plugin-sdk/sdk

SDK 公共 API 与 CLI 虽职责不同，但同属一个小型独立发布包，统一入口分主题说明。覆盖双格式构建、stdout/stderr 与交互/非交互模式、manifest/权限校验、生成数据和宿主镜像测试、独立运行限制；不提供自动发布操作。

## 3. 模板迁移与 Trellis 配置

- 保留并重写 `client/frontend/`、`server/backend/` 的适用主题。
- 删除 `server/frontend/` 的通用组件/Hook/状态模板，将实际服务端页面安全规则迁入后端。
- 删除 `shared/backend/`、`shared/frontend/` 的重复模板，创建 `shared/library/`。
- 新增 `plugin-sdk/sdk/`，在 Trellis packages 登记 `plugin-sdk: { path: plugin-sdk }`。
- 将 default_package 从 `@trek/client` 改为现有包键 `client`，不改变默认面向客户端的含义。
- 不修改根 package.json，不把 SDK 加进 npm workspaces。
- 删除前完整阅读相关文件并比对初始状态，确保没有用户编写的内容；扫描任务 JSONL 和文档引用。若发现新增外部引用，先迁移或停下确认，不留下失效链接。

## 4. 证据与文档契约

每个规范索引含：适用代码路径、开发前检查（Pre-Development Checklist）、主题链接、质量检查（Quality Check）。正文使用中文，保留原始符号、API、字段和命令名称。

每项重要规则包含：

1. 适用场景或边界。
2. 当前行为或新增代码约束的明确标签。
3. 真实路径与符号、测试行为，短代码示例仅用于解释关键形状。
4. 常见错误以及会受影响的另一端。
5. 有效的验证命令；未执行的命令不写成已通过。

源文件引用用仓库相对路径并点名符号；规范间用有效 Markdown 相对链接。避免复制大段源码、固化易变计数、声称外部 MCP 服务或特定 IDE 必然可用。

已识别需纠正的叙述见 `research/repository-analysis.md`：Service Worker 不应描述为缓存业务 API；后端 build 不等于类型检查；Page 规则仅针对页面默认导出；SDK parity 在独立 checkout 下可能跳过；实际存在的历史模式不能自动当成推荐范式。

## 5. 上下文与执行模式

初始化时 spec 尚未可信，不把即将重写的模板作为实现/审查依据。`implement.jsonl` 与 `check.jsonl` 引用当前任务研究笔记，规划三件套由任务系统单独载入；源码在执行时按主题直接读取，不加入上下文清单。

默认由一个负责者逐包完成；如执行平台要求委派，也只顺序委派一个拥有完整范围的执行角色，不并行争写目录。跨包术语和最终验收仍由主负责者统一。

## 6. 验证与回退

文档任务不需要安装依赖、启动应用或全量构建。检查重点为模板残留、空段落、链接、源码路径、索引覆盖、脚本真实性、包发现输出、JSONL 引用以及变更范围。

执行前保存允许修改文件的内容基线到当前任务的本地备份位置，并记录新增路径；现有 `.trellis/` 未跟踪，不能依赖 git restore 恢复它们。回退仅恢复本任务改动且未被并行编辑的文件，移除本任务新增文件；禁止 git reset/clean 或覆盖用户已有改动。不运行可能自动提交的归档/会话收尾命令。

最终规范不触及产品运行路径。若目录调整不被当前工具发现，应暂停并保留基线，不能为适配文档顺手改 Trellis 运行时。
