# 客户端质量检查

适用范围：`client/` 产品开发。以下命令是按 `client/package.json` 核对的入口，不代表本次文档编写执行过。

## 前置条件与工作目录

根仓库安装使用根 lockfile，client 属于 npm workspace。首次检查或 shared 有改动时，从根执行 `npm run build --workspace=@trek/shared`；客户端消费 shared 导出产物，不应把缺少产物误判为前端类型错误。

下表命令均在 `client/` 执行。按改动选择，不要求每次文本/样式小改都全量构建。

## 只读源码检查

| 命令 | 检查内容与限制 |
| --- | --- |
| `npm run lint:check` / `npm run lint` | 均为 `eslint .`，没有隐含 `--fix` |
| `npm run lint:pages` | 默认导出及指定桌面/手机 Page 容器，不扫描全文件禁 Hook |
| `npm run typecheck` | `tsc --noEmit`；Vite build 不能替代 |
| `npm run format:check` | 仅检查 src 下 TSX/CSS，不代表所有 TS 已格式检查 |
| `npm run theme:lint` | 报告硬编码样式，默认不失败 |
| `npm run theme:lint:strict` | 有命中则失败；扫描仍有例外和命名 palette 类盲区 |

## 测试分层

`client/vitest.config.ts` 使用自定义 jsdom 环境、forks pool，收集 tests 目录和 src 同位测试；MSW 模拟 HTTP，fake-indexeddb 模拟 Dexie。

| 目标 | 推荐定向命令/依据 |
| --- | --- |
| 页面 wiring | `npm test -- src/pages/AtlasPage.wiring.test.tsx`，mock Hook 验证渲染连接 |
| 共享业务 Hook | `npm test -- src/components/Admin/useInstanceSettings.test.ts`，保存顺序与竞态 |
| Store/Slice | `npm test -- src/store/slices/budgetSlice.test.ts`，操作、回滚与关联刷新 |
| 离线与冲突 | `npm test -- src/repo/placeRepo.offline.test.ts tests/unit/sync/mutationQueue.conflict.test.ts` |
| WS 完整分类 | `npm test -- tests/unit/remoteEventHandler/registry-parity.test.ts` |
| 外观 | `npm test -- src/theme/applyAppearance.test.ts` |

- `npm test` 执行全部 Vitest；`test:unit` 只限定 `tests/unit`，不会覆盖全部同位测试。
- `test:integration` 脚本包含 integration 目录和 src glob；精确验证优先传实际测试文件。
- `test:coverage` 生成 coverage 文件并检查当前配置的四项 85% 阈值，不能用普通 test 成功代替。
- 旧测试中的 `window.__addToast` 夹具或吞错用例是兼容证据，不覆盖新代码禁止全局状态/静默写失败的约束。

## 会产生或改写文件的命令

- `npm run format` 使用 Prettier `--write`；不要与只读检查混淆。
- `npm run build` 先生成 PWA 图标，再 Vite 输出 dist；`build:analyze` 额外输出 `dist/stats.html`，它不是 `build` 的同名生命周期。
- `npm run check:gl-split` 只读已构建的 `dist/assets`；缺目录、完全找不到引擎标记或同 chunk 含两种标记会失败，但**仅缺其中一种引擎不一定失败**。它不是两种 renderer 功能可用性测试，仍需新鲜构建与各引擎验证。
- `npm run e2e` 启动 Playwright 配置的服务并写测试产物。`client/playwright.config.ts` 后端不复用现有服务，使用测试启动器的隔离 SQLite；先确认端口和测试环境，绝不指向真实用户库。
- `npm run shots` 采集截图；`shots:promote` 接受截图并改文件。截图项目不做流程正确性断言，也不要未确认就 promote。

## 提交前人工检查

1. 桌面和手机是否只复用逻辑、分别验证交互？Page 提取是否保持 JSX 行为？
2. API 类型、离线缓存、队列和 WS reducer 是否同字段、同权限语义？
3. HTTP 错误是否可见，乐观操作是否回滚，pending/failed/conflict 是否区别展示？
4. 主题 token、外链/Markdown 安全、错误边界和异步清理是否保留？
5. 修改地图是否检查两个 renderer、GL chunk 和预下载资源？

记录执行命令、结果与未执行原因。纯规范任务只做文档链接、证据路径和命令真实性检查，不安装、不启动、不执行产品测试构建。项目共用约束见[跨包指南](../../guides/index.md)。
