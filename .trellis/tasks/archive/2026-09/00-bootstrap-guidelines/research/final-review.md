# 最终规范质量审查

## 结论与边界

本轮作为 trellis-check 执行角色，已完整阅读任务 check.jsonl 引用、prd.md、design.md、implement.md、remaining-implementation.md，以及全部 31 份规范和索引。对高风险规则回查了真实实现、脚本及代表性测试断言，不仅检查路径存在。

发现的问题已在批准范围内修正文档。**本轮审查未发现剩余阻塞项，AC1–AC6 按文档任务标准通过。** 这不是产品测试、安全审计或跨平台运行已通过的声明。

本轮只修改下列 12 份既有规范，并新增本报告；未改 config.yaml、任务规划/状态/JSONL、备份、业务源码、测试、依赖、CI、CLAUDE、平台技能或运行时脚本。任务复选框与最终收尾留给主会话同步，不执行 finish、commit、push 或 archive。

## 本轮修复文件与事实依据

以下规范路径均相对于 `.trellis/spec/`。

| 文件 | 已修复的问题 | 直接核查的依据 |
| --- | --- | --- |
| `guides/local-development.md` | 补可移植的最小 HTTP `.env` 示例，明确合并而非覆盖；补独立终端的 Source Map 编译、inspector 启动、前端启动、cwd 和副作用；避免根 dev 重复启动后端 | 根/各包 package.json、server/.env.example、server/scripts/dev.mjs、build.mjs、tsconfig.build.json、server/src/index.ts、client/vite.config.js |
| `server/backend/api-and-auth.md` | body 启动检查依赖 `@Body` metadata，不扫描任意 req.body；公开清单只强制 Public 与匿名 guard 链，不能声称 OptionalAuth 也有独立清单门禁；Public 入口还可能自行验证另一类凭据 | validate-body-contracts.ts、validate-route-guards.ts、global-auth.guard.ts |
| `server/backend/configuration.md` | 补 managedPreconditions 的跨字段例外：启用托管安装而缺 ENCRYPTION_KEY 会拒绝启动，不能泛称任何空变量都合法 | server/src/app-config/env.ts 的 validateEnvAtBoot/managedPreconditions |
| `server/backend/logging-guidelines.md` | 宿主 audit sink 是可选的，且仅审计 isAuditable 的方法；不是所有 RPC 都必写审计，审计写入失败不阻断调用 | PluginRpcHost.dispatch 的条件与 catch；PluginSupervisor.recordLog |
| `server/backend/integrations-and-plugins.md` | 补 REST/MCP 两条权限调用链如何汇合，MCP 私有广播必须传接收者；补普通 URL 与管理员配置服务的不同 SSRF 路径；明确 MCP 广播测试仍是 mock；相对链接加 `./` 消除 Marksman 歧义 | DaysController、DaysMcp、McpToolGuardsService、TripAccessGuard、tools-days.test.ts、ssrfGuard.ts |
| `shared/library/i18n-and-sanitization.md` | 补 Node/浏览器净化与客户端 SSR 的区别；ALLOWED_ATTR 空数组不等于关闭所有属性类别；不能把 HTML 净化当 URL/SSRF/外链 rel 的完整保证 | shared/src/sanitize/sanitize.ts、sanitize.spec.ts、shared/vitest.config.ts |
| `client/frontend/type-safety.md` | 修正依赖方向的歧义：禁止 shared 反向导入客户端模块，而不是禁止客户端正常导入 shared | shared/CLAUDE.md、客户端 API/shared 导入与浏览器状态入口 |
| `client/frontend/offline-sync.md` | 明确 flush 锁与临时 ID 只在当前运行实例/会话内保证；联网入口检查不等于逐项重查；列出可重试 HTTP 状态；区分 pending 查询包含 syncing 与回放只选 pending | mutationQueue.ts 全文、placeRepo.ts、authStore.logout、冲突测试 |
| `client/frontend/theme-and-maps.md` | 补 style 文档优先走 gl-map-styles，与预取器 gl-map-offline 不同；收紧 GL 分包门禁的保证 | Vite runtimeCaching 顺序、glPrefetcher.ts、check-gl-split.mjs |
| `client/frontend/quality-guidelines.md` | 修正“缺引擎会失败”：分包脚本仅在两种标记均缺失或同 chunk 混合时失败，不能保证两个引擎都存在 | client/scripts/check-gl-split.mjs |
| `plugin-sdk/sdk/api-and-cli.md` | 将“跳过 symlink”限定为 walk 递归分支；根文件单独 statSync，不能误称全打包路径具有 realpath containment | plugin-sdk/src/cli/pack.ts |
| `plugin-sdk/sdk/host-parity.md` | 明确 registry skip 只检查 validate-entry.mjs；它存在但 check-readme.mjs 缺失时会读取失败，不是自动 skip | plugin-sdk/test/checks-parity.test.ts |

这些修正描述当前行为和验证边界，没有借文档任务改产品实现或宣称修复了历史运行时债务。

## 独立内容复核范围

长源码按主题读取相关片段；以下列出本轮实际复核的主要证据，不声称每个长文件均全文审计。

### 共享指南、服务端与 shared（前序 18 份）

- 完整核对根和四个包的 package.json。确认 shared 先构建、SDK 独立安装、根 test 不含 SDK 而 test:cov 显式追加 SDK；server/shared lint 有 fix，client lint 没有；format glob 的真实覆盖范围没有被夸大。
- 阅读 server/src/index.ts、bootstrap.ts、AppModule：配置校验副作用顺序、单一 HTTP server 与 WS 绑定、pre-init middleware、命名 parser、MCP 原始 stream、三类启动检查及全局 guard 顺序。
- 阅读 GlobalAuthGuard、jwt-verify.ts、TripAccessGuard、Zod pipe、body/route validators；对照 trip-access.guard.test.ts 的 TRIPGUARD-002 与相关拒绝行为。读取 CollectionsService 的绑定查询、可见性与编辑权限分支，不把请求 Schema 当资源归属校验。
- 阅读 DaysController、DaysMcp、McpToolGuardsService；对照 tools-days.test.ts 的 title/notes 独立更新、实际读回、跨 trip/非成员拒绝、广播断言。scope、资源可见性、动作权限与广播接收者是不同检查。
- 阅读 database.ts、DatabaseService、seeds.ts、config.ts；读取 runMigrations 尾部执行循环及 migration-version-atomicity.test.ts。确认同步事务、普通迁移与 schema_version 同事务、raw 例外、测试内存库、首次管理员与密钥读写边界。
- 阅读 BudgetService 的 splitEqualShares、calculateSettlement 与 toTripCents：负金额 floor/余数、费用自身币种到行程整数分、冻结汇率优先与显示币种转换；没有把线上 amount 一概写成 cents。
- 阅读 app-config/env.ts 与 validate.test.ts；读取日志模块、redact 与 redaction.test.ts；对照异常过滤器的原生 error 透传、普通 5xx 隐藏、Multer 和 headersSent 分支。
- 阅读 StorageService 的门面、sendToResponse/流分支及 storage-keys.test.ts；CronRegistrarService 的测试 gate 与关闭清理；SSRF 的普通与管理员路径、逐跳重校验和凭据处理。
- 阅读 PluginRpcHost、PluginSupervisor 的 spawn/acting user/log limiter、PluginFrameController 及 plugin-frame.test.ts：权限注册、IPC 上下文、环境白名单、CSP、CORP、词法与 realpath containment。符号链接测试依赖平台能力，不能用源码阅读替代真实平台运行。
- 阅读 shared primitives、collection 输入/输出差异、weather.schema.spec.ts、datetime-normalize.ts、语言注册表、i18n parity 脚本/测试与 placeholder 测试、sanitize 实现/测试、tsdown 与 exports。普通 parity 退出 0、缺译文不在 placeholder 测试覆盖中、消费者读取 dist 等约束成立。

### 客户端与 SDK（其余 13 份）

- 对照 client/CLAUDE.md、PATTERN.md、check-page-pattern.mjs 与 useInstanceSettings 实现/测试，确认 Page/业务 Hook/Model 分工、Desktop/Mobile 容器的额外扫描、旧例外与异步动作隔离，不将子组件局部状态误判为违规。
- 完整读取 mutationQueue.ts、placeRepo.ts、onlineThenCache；读取 offlineDb、authStore.logout、remoteEventHandler 相关片段与 registry-parity、placeRepo.offline、mutationQueue.conflict 测试。确认稳定 UUID、负 ID 依赖持久化改写、版本令牌推进、三种冲突策略、HTTP 错误不退缓存、私有 packing 缓存删除及事件唯一分类。
- 阅读 API dev parse、applyAppearance 与测试、MapViewAuto/glLazy、glPrefetcher、Vite 缓存规则、theme-lint/check-gl-split。dev parse 只是警告；公开页并未清掉全部字号/密度；地图预取与缓存命中不是同一保证。
- 核对客户端 Vitest 与 Playwright 配置：同位测试、85% 四项阈值、后端不复用的测试启动器；没有将普通 test、浏览器 E2E、截图或构建互相替代。
- 核对 SDK 双格式 tsconfig 与 finish-cjs、CLI pack/checks/ui/status、manifest settings/action 默认 scope、permissions 与 grantGaps；读取 permissions-parity、checks-parity、mock-limits。生成器确实同时读取 envelope 和 plugin-event-sink，并写 SDK/shared；parity 只覆盖指定集合，独立 checkout/registry 缺失不能记作宿主一致性通过。
- 前轮 remaining-implementation.md 的其余源码/测试证据用于交叉对照，未将其记录的未执行产品命令冒充本轮执行结果。

## 实际执行的验证

### 命令

```bash
python3 ./.trellis/scripts/get_context.py --mode packages
python3 ./.trellis/scripts/task.py validate .trellis/tasks/00-bootstrap-guidelines
python3 ./.trellis/scripts/task.py list-context .trellis/tasks/00-bootstrap-guidelines
rg -n 'To be filled by the team|To fill|TODO: fill|Fill in each file|your-project|path/to/' .trellis/spec
git diff -- .gitattributes
git diff --cached --stat
git diff --check
git status --short
```

- 包发现列出 client/frontend（默认）、server/backend、shared/library、plugin-sdk/sdk 与 guides。
- task validate/list-context 通过；两个 JSONL 各有一个真实研究引用，未引用删除层。
- 占位符搜索无匹配，退出码 1 是预期结果。
- git diff --check 通过；Git 当前仅跟踪到既有 `.gitattributes` 改动，因此另外检查未跟踪规范。

### Python 只读检查（以 heredoc 运行，未创建脚本文件）

- 31 份 Markdown：guides 5、client/frontend 9、server/backend 9、shared/library 4、plugin-sdk/sdk 4。
- 83 个本地 Markdown 链接按文档目录解析有效；片段检查无问题，所有主题可从 guides 索引到达，所有非 index 主题均由同目录 index 导航。
- 无模板残留、空叶子标题、行尾空白或缺少末尾换行。索引全部包含 Pre-Development Checklist 与 Quality Check。
- 175 个去重的字面源码/配置路径存在。`server/.env`、运行时数据库 `server/data/travel.db` 和调试产物 glob `server/dist/**/*.js` 按运行时/模式引用处理，不为了通过检查创建它们。
- 抽取明确 npm 命令，核对 39 个包/脚本组合与 29 个定向测试路径，无不存在项。没有执行这些 npm 命令。
- 扫描全部任务 JSONL：2 个文件、2 个真实 file 引用，没有目录迁移断链。
- 对照 `.baseline`：原 36 份规范变为 31 份，删除 20、新增 15；删除层为 server/frontend、shared/backend、shared/frontend。删除前完整阅读及逐字节基线比较已由前轮执行角色记录，本轮不重复恢复/删除文件。
- `.baseline/tracked-hashes.json` 的 4269 个已跟踪文件 SHA256 全部一致，包含既有 `.gitattributes` 修改；git status 与基线一致。
- config.yaml 与基线做预期两处替换后全文一致：只增加 plugin-sdk 映射、default_package 改 client。本轮未编辑配置。

### 自动诊断与限制

修改后的规范由编辑工具返回 Markdown clean。曾出现一条 Marksman 同名 `directory-structure.md` 链接歧义，改为 `./directory-structure.md` 后，最新 turn_delta 诊断为 0 warning；这不冒充全仓 LSP 扫描。

config.yaml 第 144–146 行超长注释是前轮已确认的基线告警，全文基线比较也证明本轮未改变它们；按授权保持原样，不列为阻塞项。没有为此改平台、运行时或配置格式。

## AC1–AC6

| 验收项 | 结果 | 依据 |
| --- | --- | --- |
| AC1 职责与发现 | 通过 | 四包有效层与真实职责一致；包发现成功，错误空模板层已移除 |
| AC2 可追溯与高风险契约 | 通过 | 全文审查并回查上述源码/测试；修正过强保证，保留当前行为、新代码约束和债务区别 |
| AC3 完整性与链接 | 通过 | 31 份规范、83 个有效本地链接、索引全覆盖；无模板/空叶子段落 |
| AC4 运行与验证入口 | 通过 | 脚本匹配；shared 构建顺序、SDK 独立性、build/typecheck 区别、TS 调试和副作用明确 |
| AC5 任务上下文 | 通过 | 包发现、task validate/list-context 及所有任务 JSONL 路径检查通过 |
| AC6 范围与如实记录 | 通过 | tracked 哈希未变，配置仅原许可映射；本轮只写规范与本报告，未执行产品验证 |

## 未执行项与交接

没有安装依赖、启动应用、运行 npm lint/typecheck/test/build、浏览器/地图测试、生成器、外部索引、网络上传、registry preflight、发布、数据库或账号操作，也没有 commit/push/archive/finish。文档中的调试步骤只做静态核对，未实际 attach 或跨平台试跑。

当前无待修复的文档阻塞项。产品历史限制（跨标签页锁/ID、在途账号切换、GL 缓存分流、guard 清单覆盖、SDK mock/skip 等）均按真实保证记录，不在本任务扩修。主会话仍应执行其约定的最终结构/范围检查，并同步任务进度；本轮不替主会话修改任务状态。
