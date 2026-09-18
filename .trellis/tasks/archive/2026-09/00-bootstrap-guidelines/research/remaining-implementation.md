# 剩余规范实施记录

## 本轮范围与结果

本轮以 trellis-implement 执行角色续做已批准任务，先读取 implement.jsonl 引用、PRD、design、implement；未重新委派、未修改任务阶段或既有规划。

- 完成 `client/frontend/` 9 份规范：重写 7 份原模板，新增 offline-sync、theme-and-maps。
- 新增 `plugin-sdk/sdk/` 4 份规范：index、api-and-cli、host-parity、quality-guidelines。
- 合计交付 13 份正文，均为中文、约 41–63 行，含真实源码/测试依据、当前实现与新代码约束、误区和验证入口。
- 删除 `server/frontend/` 7 份、`shared/backend/` 6 份、`shared/frontend/` 7 份，共 20 份不适用模板；删除前已逐份完整阅读并与原始备份逐字节比较，全部仍为初始模板，没有待迁移的用户正文。
- `.trellis/config.yaml` 仅新增 plugin-sdk path、将 default_package 改为 client；与基线做预期替换后全文一致校验通过。
- 本文件为本轮唯一新增任务研究产物。前序 guides 5 份、server/backend 9 份、shared/library 4 份，以及其他任务文档均未改写。
- 本轮涉及 35 个路径：13 份规范写入、20 份模板删除、1 份配置修改、1 份本记录。最终规范树共 31 份 Markdown。

## 客户端实际阅读证据

以下为实际直接读取的依据，不表示完成整个客户端审计。标注“片段”的长文件仅按相关主题阅读，没有宣称全文核查；规范中的未来验证命令没有在本轮执行。

### 约定、配置与页面

- 完整读取 `client/CLAUDE.md`、`client/src/pages/PATTERN.md`、`client/package.json`、`client/vitest.config.ts`、`client/playwright.config.ts`。
- `client/src/App.tsx` 前 180 行：懒加载、桌面/手机入口、ProtectedRoute、路由错误边界。
- `client/src/pages/TripPlannerPage.tsx` 前 180 行：LazyPanel、子组件 ListsContainer 的局部状态；不能把 Page 限制套到整个文件。
- 完整读取 `client/scripts/check-page-pattern.mjs`：除了默认导出，也检查顶层 PageDesktop/PageMobile 容器，存在限定 KNOWN_ESCAPES。
- `client/src/pages/atlas/useAtlas.ts` 前 180 行与 `atlasModel.ts` 前 100 行：地图状态/实例、取消引用、共享纯派生和历史查找表。
- `client/src/pages/AtlasPage.wiring.test.tsx` 前 150 行：mock Hook、加载展示、标记交互、服务错误 toast；没有将 wiring 当成全部业务测试。
- `client/src/pages/admin/AdminSettingsTab.tsx` 前 90 行与 `client/src/mobile/screens/admin/MAdminSettingsSection.tsx` 前 80 行：共享 useAdmin 结果，但仍有历史直接请求/重复 handler。
- 完整读取 `client/src/components/Admin/useInstanceSettings.ts`，并读同名测试前 120 行：脏表单保存、danger 确认、正在保存的新编辑保留、旧动作结果隔离。

### 状态、缓存、同步与安全

- `client/src/store/tripStore.ts` 前 160 行：领域切片、resetTrip、并发加载及非致命读取降级。
- `client/src/store/slices/budgetSlice.test.ts` 前 120 行：加载、写入错误、删除回滚、关联 reservation 刷新，也记录旧 load 吞错。
- 完整读取 `client/src/repo/placeRepo.ts`、`client/src/repo/withOfflineFallback.ts`、`client/src/sync/networkMode.ts`、`client/src/sync/authGate.ts`、`client/src/hooks/useNetworkMode.ts`。
- 完整读取 `client/src/sync/mutationQueue.ts`：负 ID/UUID、pending FIFO、stuck syncing 恢复、稳定幂等键、依赖 ID 持久化改写、版本推进、409 策略、终止/重试区分。
- `client/src/db/offlineDb.ts` 前 240 行：用户库命名、Proxy 切库、队列字段和 Dexie v1–v4；同名测试前 100 行核对测试环境与 upsert 入口。
- `client/src/repo/placeRepo.offline.test.ts` 前 140 行；`client/tests/unit/sync/mutationQueue.test.ts` 前 100 行；`mutationQueue.conflict.test.ts` 前 150 行：负 ID、trip_id、header、冲突停放/解决、连续更新令牌的测试入口。
- `client/src/store/slices/remoteEventHandler.ts` 前 240 行：STATE_APPLIERS、DEXIE_WRITERS、更新后 Day 回填及私有 packing 删除缓存。
- 完整读取 `client/tests/unit/remoteEventHandler/registry-parity.test.ts`：事件唯一分类、无陈旧名称、trip scope、writer 必有 applier。
- 完整读取 `client/src/sync/tripSyncManager.ts`：bundle、缓存选择、文件排除、prepare 与后台同步边界；未声称所有在途账号切换都已彻底防护。
- `client/src/api/client.ts` 前 160 行与 `client/src/types.ts` 前 130 行：shared 类型、dev-only 校验、统一 Axios、幂等头与仍存本地接口。
- 完整读取 `client/src/components/shared/markdownSanitize.tsx`；`client/src/i18n/TranslationContext.tsx` 前 130 行：raw-as-text、净化、外链、语言动态加载与取消保护。

### 主题、地图和插件 frame

- 完整读取 `client/src/theme/README.md`、`client/src/theme/applyAppearance.ts`；同名测试前 130 行：主题标记、自定义 accent、共享页中性规则。
- `client/tailwind.config.js` 前 100 行；完整读取 `client/scripts/theme-lint.mjs`、`client/scripts/check-gl-split.mjs`：语义/手机 token、扫描盲区、默认退出码、构建产物检查。
- 完整读取 `client/src/components/Map/MapViewAuto.tsx`、`client/src/components/Map/glLazy.tsx`：fallback、引擎注入、分包。
- `client/vite.config.js` 前 200 行：API NetworkOnly、壳缓存、Mapbox/OpenFreeMap 的不同缓存规则。
- 完整读取 `client/src/sync/glPrefetcher.ts`；`tilePrefetcher.ts` 1–180 与 255–464 行：vector 模板分流、cache 名、CORS/no-cors、容量和持久化前提。
- `client/src/components/Plugins/PluginFrame.tsx` 的 token 读取和尾部 iframe/observer 清理片段：全局与 m-root 作用域、导航后桥接停止、sandbox 和 referrerPolicy。

## SDK 实际阅读证据

### 公共面、构建与 CLI

- 完整读取 `plugin-sdk/CLAUDE.md`、`plugin-sdk/package.json`、`plugin-sdk/tsconfig.json`、`plugin-sdk/tsconfig.cjs.json`、`plugin-sdk/vitest.config.ts`、`plugin-sdk/scripts/finish-cjs.mjs`。
- `plugin-sdk/src/index.ts` 前 180 行与尾部 definePlugin/导出/session 片段：作者实体开放字段、ctx、acting user、instance/user scope、对象原样返回。
- `plugin-sdk/src/manifest.ts` 前 330 行：类型、settingDefaults、KNOWN_ADDONS、可满足版本范围、权限/egress、capability 校验。
- 完整读取 `plugin-sdk/src/permissions.ts` 与 `plugin-sdk/src/cli/checks/index.ts`：非 ctx 入口授权、grantedHosts、同步 offline 与异步 all 检查。
- `plugin-sdk/src/cli/trek-plugin.ts` 参数/路由/help/status 片段；`plugin-sdk/src/cli/ui.ts` 前 110 行；完整读取 validate/status/update-notice 模块：未知 flag、TTY、stderr、退出码职责。
- `plugin-sdk/src/cli/entry.ts` 前 180 行；完整读取 `plugin-sdk/src/cli/pack.ts`：真实产物 hash/size、依赖声明、签名拒绝降级、artifact 与 publish gate 分离。
- `plugin-sdk/src/cli/dev.ts` 前 100 行、`plugin-sdk/src/ui/kit.ts` 前 100 行：开发模拟、可选 SQLite、内联 UI kit 并非隔离边界。

### 镜像与测试

- `plugin-sdk/src/mock-host.ts` 前 150 行：fixture、成员/应用权限、Addon、声明与 daily-budget 生命周期限制。
- `plugin-sdk/src/egress-policy.ts` 前 130 行：纯 helper、私网拒绝、dev 不复制全部宿主加固。
- 完整读取 `server/scripts/gen-plugin-facts.ts`、`plugin-sdk/scripts/gen-lucide-icon-names.mjs`；`plugin-sdk/src/generated/host-facts.ts` 前 90 行：两类源、两份输出、覆盖检查、宽类型与提交快照。
- 完整读取 `plugin-sdk/test/permissions-parity.test.ts`、`plugin-sdk/test/permissions.test.ts`、`plugin-sdk/test/manifest-roundtrip.test.ts`、`plugin-sdk/test/manifest-settings.test.ts`、`plugin-sdk/test/mock-limits.test.ts`。
- `plugin-sdk/test/checks-parity.test.ts` 前 140 行：registry 路径条件、skip、SKIP_NETWORK 及真实脚本调用。
- `plugin-sdk/test/sdk.test.ts` 前 180 行；`plugin-sdk/test/cli.test.ts` 前 180 行：对象身份、成员拒绝、session scope、egress 脚手架、权限全集与 TTY。
- `plugin-sdk/test/ui-kit.test.ts` 前 100 行、`plugin-sdk/test/checks.test.ts` 前 100 行、`plugin-sdk/test/egress-policy.test.ts` 前 90 行：内联安全/消息源、未知 icon 阻断、出站匹配及地址拒绝。
- 完整读取根 `package.json`，并读取 `server/package.json` 脚本段，核对 workspace、test:cov 显式包含 SDK、生成/check 命令。

## 纠正的历史叙述

1. Page 检查不是全文件禁 Hook，也不只看默认导出；当前脚本还看指定 Desktop/Mobile 容器。
2. API 的 dev parse 只警告、不强制拒绝；SW 不缓存业务 API。
3. OpenFreeMap 有矢量资源预下载；入口按 tile 模板分流，不代表任意 GL provider 都有同样能力。
4. 新代码禁止绕层/静默写错，不代表现存组件/Hook/测试已全部符合。
5. SDK egress parity 只比较列出的 helper 并归一化空白，不是整个文件逐字节验证。
6. host-facts 的事件来源还包括 plugin-event-sink，不只有 envelope。
7. SDK 当前 registry 离线检查把未知 icon 判为失败；“只 warning”的生成器/约定文字已过时。
8. SDK mock budget 不自动跨午夜重置；独立 checkout 的 parity skip 不等价于宿主验证通过。

## 文档自检结果

已实际执行：

- `python3 ./.trellis/scripts/get_context.py --mode packages`：显示 client/frontend（默认）、server/backend、shared/library、plugin-sdk/sdk。
- `python3 ./.trellis/scripts/task.py validate .trellis/tasks/00-bootstrap-guidelines`：implement/check 均有 1 个真实研究引用且通过。
- `python3 ./.trellis/scripts/task.py list-context .trellis/tasks/00-bootstrap-guidelines`：仍引用 repository-analysis，无目录迁移断链。
- 占位符搜索：无匹配，rg 返回 1，符合预期；不是执行失败。
- Python 只读文档检查：31 份规范，83 个本地 Markdown 链接，标题片段/文件存在、索引全覆盖、无空叶子段落、无行尾空白通过；仓库引用排除 glob/示意路径后逐项检查。
- 初版检查把“标题后直接进入有正文的子节”和运行时 `server/data/travel.db` 误判；修正判定后通过，未为误报修改前序规范或创建数据库。
- `git diff --check` 通过；额外检查未跟踪规范，避免 Git 不显示新文件导致漏检。
- 4269 个已跟踪文件 SHA256 全部与任务基线一致；`.gitattributes` 的既有改动保留，git status 形状与基线一致。
- 配置全文只存在获准的包映射变更；未覆盖 `.baseline/`，未修改产品代码、测试、依赖、CI、CLAUDE、技能或运行时。

## 遗留诊断与未执行项

自动 YAML 检查报告 `.trellis/config.yaml` 第 144–146 行的 3 条注释超过 80 字符。已直接读取并与 `.baseline/config.yaml` 对比，三行完全相同：这是既有格式诊断，不是新增映射引入的语法/包发现问题。因本轮明确只允许改 packages/default_package，未擅自格式化这些注释；交主会话决定是否需要单独授权处理，不能写成所有自动诊断均已清零。

本轮没有执行 npm 安装、产品 lint/typecheck/test/build、浏览器/地图验证、生成器、联网 registry 检查、发布、数据库操作或 commit/push/archive/finish。规范中记载的未来命令均未冒充执行结果。没有新增阻塞包发现、JSONL 或文档链接的问题；前序 18 份规范的最终内容复审与任务整体 AC 验收由主会话统一完成。
