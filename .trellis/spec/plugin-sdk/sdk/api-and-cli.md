# 公共 API 与 CLI

适用范围：`plugin-sdk/src/` 的作者 API、manifest 和命令接口。

## 当前实现：API 与导出

- `plugin-sdk/src/index.ts` 的 `definePlugin(def)` 原样返回 def；`PLUGIN_API_VERSION` 表示插件 API 兼容级别，破坏性变化按包约定评估升级。
- `PluginContext` 是宿主注入能力。Trip/Place 等类型只保证 id，许多字段可选并保留 unknown 索引；不能当成应用 REST Schema 的完整替代品。
- trip 访问由 acting user 决定，兼容参数 `asUserId` 不能冒充用户；后台 job/onLoad 没有同样的用户上下文。
- `ctx.config` 是激活时的实例设置快照，`ctx.settings.get` 读取当前执行用户的设置；不要把 instance/user scope 混用。
- `plugin-sdk/test/sdk.test.ts` 验证 definePlugin 对象身份、无权限拒绝和 acting user 的成员校验。

## 当前实现：独立双格式包

`plugin-sdk/package.json` 的 `.` 和 `./testing` 都有 ESM/CJS 入口与 types。构建顺序为主 tsconfig → CJS tsconfig → `scripts/finish-cjs.mjs`。

主构建为 NodeNext/ES2022；CJS 配置以 index/manifest/mock-host 为入口，不把整个 CLI 再构建一套。`finish-cjs.mjs` 写入 `dist/cjs/package.json` 的 commonjs 标记，否则根包 type=module 会错误解释 CJS。

**新增代码约束**：保留独立安装与 NodeNext 的 `.js` 导入写法；不跨包引用 TREK 源码，不为便捷开发改变 root workspaces。具体测试见[质量检查](quality-guidelines.md)。

## 当前实现：三层校验不能混为一谈

1. `plugin-sdk/src/manifest.ts` 的 `validateManifest` 返回错误集合和归一化 manifest，不以 throw 作为一般无效输入结果。检查 slug、版本、可满足的 trek 范围、权限、egress 和能力声明。
2. `plugin-sdk/src/cli/checks/index.ts` 的同步 `runOffline` 执行工作树检查；`runAll` 追加需要 GitHub 的检查。规则、渲染、退出码分别管理。
3. `plugin-sdk/src/cli/pack.ts` 只取 `artifact` 阻塞项；`validate` 执行全部离线 gate。能打 ZIP 不等于能发布到 registry。

manifest 的 settings 默认 scope 是 instance，actions 默认 scope 是 user；`settingDefaults` 忽略 secret 默认值。新增字段同时核对宿主归一化与 `plugin-sdk/test/manifest-roundtrip.test.ts`，防止验证接受却在输出中丢字段。

`packPluginDir` 收集允许的根文件及 server/client 内容；`walk` 递归时跳过 `.ts`/`.map`、依赖树和 symlink，并对产物检查 native binary、大小和条目数。根文件收集另走 `statSync`，不能把递归跳过链接泛化为整个打包路径的 realpath containment 保证；宿主安装还有独立检查。不要用普通目录压缩替代此产物契约。

**历史说明纠偏**：`plugin-sdk/test/checks.test.ts` 当前明确断言未知 Lucide icon 导致 `manifest.icon` 失败。包级说明与图标生成器里“只 warning”的旧描述不能代替实际 CLI gate；manifest 本身通过也不代表目录 validate 通过。

## 新增代码约束：CLI 协议

依据 `plugin-sdk/CLAUDE.md` 和 `plugin-sdk/src/cli/ui.ts`：

- `isInteractive` 要求 stdin/stdout 均为 TTY；非交互调用必须由参数驱动，不挂起等待输入。
- JSON、artifact 结果、PR URL 等机器输出走 stdout；装饰、提示、进度和更新通知走 stderr。并非所有 stdout 都是 JSON，help/status 仍输出文本。
- `trek-plugin.ts` 的 `COMMAND_FLAGS` 拒绝未知 flag；新增参数同步帮助，不静默忽略拼错的选项。
- 正常 `status` 用于指引，不以检查失败返回非零；`validate` 才是阻断 gate。CLI 参数错误仍可失败。
- 更新提醒是 advisory；不能让网络或缓存故障破坏主命令或污染机器输出。

`plugin-sdk/src/cli/entry.ts` 的 `buildEntry` 从真实 ZIP 算 sha256/size，保留 trek 范围及依赖声明；不要手写派生元数据或在签名后改变 ZIP 字节。

## 权限与本地开发限制

`plugin-sdk/src/permissions.ts` 的 `grantGaps` 检查 hooks/events/jobs/GDPR 入口缺少授权；生产可能静默不投递，dev/mock 故意让它更明显，不应为“测试方便”删除拒绝。

`grantedHosts` 只从 `http:outbound:<host>` 提取运行期目标；egress 数组是声明，裸权限不等于任意网络。operatorEgress 也需管理员配置目标，不是 allow-all。

`plugin-sdk/src/cli/dev.ts` 在 CLI 进程加载作者代码并模拟宿主，只有运行时具备 node:sqlite 才能使用真实 SQLite；它不是运行恶意插件的生产安全边界。UI kit 也不提供额外权限。

## 验证

在 `plugin-sdk/` 运行 `npm test -- test/sdk.test.ts test/cli.test.ts test/permissions.test.ts test/manifest-roundtrip.test.ts` 与 `npm run typecheck`。公共导出变化追加 build；不执行发布命令来验证普通开发修改。
