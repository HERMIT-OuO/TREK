# 规范初始化：源码分析笔记

## 分析范围与证据等级

本轮为规划阶段，复用当前会话对 README、根/各包 package.json、前端路由、tripStore、tripRepo、服务启动与开发配置的直接阅读，再核实 Trellis 目录和代表性源码/测试。没有执行产品测试、启动服务或安装依赖；本文不是全仓库审计结果。

- 已直接核对的事实可以进入规划依据。
- 包级约定文档可说明项目方向，但写入正式规范前仍需按主题补读源码和测试。
- 重要条款分别标注“当前实现”“新增代码约束”“历史兼容例外”，避免把旧代码特例泛化。

## 现有 Trellis 状态

- 初次检查时任务只有 task.json 和模板 prd.md，尚无 design.md、implement.md、JSONL 上下文；初始化记录为 in_progress，但本会话没有 active task。
- 用户授权复用该任务进入规划；已将记录状态纠正为 planning。尚未执行 task.py start。
- `.trellis/spec/` 的五个包层索引全部包含待填写项目，主题文件也可搜索到模板填充标记；guides 是通用说明，尚非 TREK 专用。
- `.trellis/config.yaml` 只登记 client、server、shared；default_package 写为 @trek/client，与 packages 的键 client 不一致。
- `.trellis/scripts/common/packages_context.py` 的 `_scan_spec_layers` 直接扫描包下子目录，不把层名硬编码为 frontend/backend，因此可用 shared/library 和 plugin-sdk/sdk。
- 同文件 `get_packages_info` 将包键与 default_package 直接比较；建议把 default_package 改为 client，保留默认面向客户端的意图。
- 配置中没有启用 registry.spec；本轮不修改模板哈希、运行时脚本或平台集成。
- 初查任务目录未发现现有 JSONL 清单；执行时再次扫描，避免覆盖并行新增的引用。

## 包与主题证据

| 包/主题 | 已阅读的依据 | 规划结论 |
| --- | --- | --- |
| 根工作区 | package.json、CLAUDE.md、README.md | 三个 npm workspace；shared 必须先构建；独立 plugin-sdk 不应加进 npm workspaces |
| 客户端页面 | client/src/App.tsx、client/src/pages/TripPlannerPage.tsx、client/src/pages/PATTERN.md | 桌面/手机视图分流；Page 默认导出为装配容器，允许同文件子组件有自己的状态；不能写成“整个文件禁止 Hook” |
| 客户端页面测试 | client/src/pages/AtlasPage.wiring.test.tsx 前部 | Mock useAtlas 后验证页面连接、交互和错误提示；测试夹具中的旧全局写法不是新增代码推荐模式 |
| 行程数据 | client/src/store/tripStore.ts、client/src/repo/tripRepo.ts、client/CLAUDE.md | Store 组合领域切片，Repository 读取网络或 Dexie；仓库仍存在直接 API 调用，不能声称所有领域均完全离线化 |
| PWA | client/vite.config.js API runtimeCaching 配置 | API 匹配项使用 NetworkOnly；离线业务数据由 repo/Dexie 管理，不能照搬根 CLAUDE 的“缓存 API”表述 |
| 服务端组织 | server/src/nest/app.module.ts、server/src/index.ts、server/CLAUDE.md | Nest 注册全部领域与全局 guard/filter/interceptor/pipe；Express 是适配器，不再是并行旧业务服务 |
| 模块/兼容示例 | server/src/nest/weather/weather.controller.ts、weather.service.ts | Controller 依赖注入；保留历史错误文本和查询参数；service 包装 impl 是当前特例，不应鼓励所有新领域复制旧模式 |
| 契约与测试 | shared/src/weather/weather.schema.ts、weather.schema.spec.ts；server/tests/unit/nest/weather.controller.test.ts | 字符串经纬度、默认 de、精确错误消息有测试依据；测试 Mock 不应被误认为生产边界类型安全规范 |
| 默认数据 | server/src/db/database.ts、seeds.ts 的相关片段；server/src/config.ts | SQLite 默认 travel.db、测试隔离内存库；首次管理员和密钥自动生成，不将固定凭据写入规范 |
| 开发调试 | server/scripts/dev.mjs、build.mjs、tsconfig.json、tsconfig.build.json；server/.env.example；wiki/Development-environment.md | build 容忍 tsc 错误；构建配置 sourceMap=false 覆盖基础配置；本地 HTTP 需调整 Cookie/示例 OIDC |
| SDK | plugin-sdk/package.json、CLAUDE.md、README.md 前部 | 独立锁文件；ESM/CJS 双导出；CLI 与 SDK 在同一发布包，不必人为分多个任务 |
| SDK 协议测试 | plugin-sdk/test/permissions-parity.test.ts | 宿主代码与 SDK 有生成数据和手工镜像；独立 checkout 下部分 parity 测试会 skip，不能描述为任意环境都有等价保证 |

表内 weather.controller.ts 和 weather.service.ts 的完整路径分别为 `server/src/nest/weather/weather.controller.ts`、`server/src/nest/weather/weather.service.ts`；其余缩写均以前一同目录路径为准。

## 需在编写规范时补充的直接阅读

1. 前端：mutationQueue、offlineDb、remoteEventHandler、相应离线/冲突测试，theme 实现与校验脚本、地图渲染器分块，页面 Hook 及桌面/手机共用逻辑实例。
2. 后端：collections 等带 Zod DTO 的写入控制器；bootstrap 与公共路由校验；SQLite 迁移/事务；日志脱敏与异常过滤；配置 fail-fast；Realtime、MCP、插件与存储边界的代表测试。
3. shared：common primitives、不同请求/响应 Schema、tsdown 导出、i18n 注册表及一致性测试、sanitize。
4. SDK：manifest、permissions、mock-host、CLI 命令/check registry、双构建收尾脚本及相关测试。
5. 每个关键模式争取对照 2–3 个源码/测试示例；若仅有特例，限制适用范围，不把它上升为统一规则。

## 工具与限制

- MCP 工具搜索 gitnexus/abcoder 没有匹配工具，本轮不安装、连接外部索引或上传代码。
- 当前会话 pi-lens 的项目图因文件数超过默认上限不可用；文件级 outline 与直接读取仍可使用。不为文档初始化改动项目索引配置。
- 采用路径搜索、文件级结构和源码/测试直读；这些工具限制不阻塞规范编写。
- 仓库约定文件偶有历史叙述冲突，优先级为可核实的当前实现/配置/测试，再参考局部约定，最后参考根概览。现有约定中明确针对新代码的要求需单独保留，不以旧代码抵消。
