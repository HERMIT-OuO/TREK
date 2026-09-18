# 日志与脱敏

## 使用现有日志入口

**当前实现**：`server/src/nest/audit/audit-log.logger.ts` 提供 `logInfo/logDebug/logWarn/logError`，写文本控制台与轮转文件，不是全项目统一 JSON logger。文件目录为运行 cwd 下 `data/logs`，首次写入才建目录；按正常 workspace 命令启动时 cwd 为 server。不要把启动 banner 中容器路径当成本地固定位置。

LOG_LEVEL 在模块导入时冻结，TZ 在格式化时读取；这是带测试时序的兼容行为，不能无依据改成每次重新设置阈值。普通业务变化记录必要的动作/资源标识，排障细节用 debug，可恢复异常用 warn，失败用 error；不要通过打印完整请求对象增加“可观测性”。

## 请求日志脱敏不覆盖任意字符串

`server/src/middleware/globalMiddleware.ts` 的 `redact` 递归处理对象/数组：敏感键不区分大小写，`_token/_key/_secret/_password/_pass/_verifier` 后缀受保护，`{ key: 'carto_api_key', value: ... }` 会按 key 隐去 value。

`server/tests/unit/middleware/globalMiddleware.redaction.test.ts` 覆盖 S3 secretAccessKey、嵌套对象数组、OAuth refresh_token/code_verifier、设置键值结构，并确认 accessKeyId/tokenCount 不被误删。

**新增代码约束**：新增密钥字段时同步核验脱敏规则与日志调用。不要打印 Cookie/Authorization、密码、验证码、私有文档内容；不要先把对象 stringify 再传给 redact，原始字符串不会被它识别。自定义日志和 Error.message 不自动经过请求脱敏。

## 审计不是普通调试日志

`server/src/nest/plugins/host/rpc-host.ts` 的 `PluginRpcHost.dispatch` 在配置了 audit sink 且 `isAuditable(method)` 为真时，记录 pluginId、actingUserId、method、resource 和结果 code，包括这些方法的拒绝；这不是所有 RPC/未知方法都必写审计的保证，审计写入失败也不会阻断调用。插件 stdout/stderr 不能作为可靠授权审计。`server/src/nest/plugins/supervisor/plugin-supervisor.ts` 的 `recordLog` 有逐插件限流与丢弃汇总，不能删掉限流让恶意日志阻塞宿主。

**历史例外**：`server/src/db/seeds.ts` 首次无用户时输出引导管理员凭据。这是受限启动流程，非日常日志范式；不能把该日志上传、复制到规范或用它证明可以记录密码。

## 验证

根目录运行 `npm run test --workspace=server -- tests/unit/middleware/globalMiddleware.redaction.test.ts`。新增日志需人工检查正常与失败路径均不含秘密，并覆盖新字段的大小写、嵌套、键值形式。纯日志门禁通过不代表其他 console.error 已完全脱敏。
