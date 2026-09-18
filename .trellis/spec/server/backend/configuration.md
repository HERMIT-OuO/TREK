# 配置、密钥与运行时语义

## 配置的两个时间点

**当前实现**：`server/src/app-config/env.ts`：

- `readEnv()` 每次从当前环境派生类型化配置，不缓存，也不每次做 Zod 校验。
- `validateEnvAtBoot()` 聚合已设置但格式非法的变量，普通字段空/未设置走默认；由 `server/src/app-config/boot-validate.ts` 导入副作用执行。另有 `managedPreconditions()` 跨字段校验：`TREK_MANAGED` 启用时缺少 `ENCRYPTION_KEY` 也拒绝启动，不能把“空值走默认”推广为所有组合都合法。
- `server/src/index.ts` 在 dotenv 后立即导入 boot-validate，早于密钥和数据库的模块副作用。不要把它换成 index 函数体里的调用，也不要移到 `buildApp()` / ConfigModule，使环境可变测试意外受限。

**新增代码约束**（`server/CLAUDE.md`）：Nest 类使用 `server/src/nest/app-config/` 的启动稳定 namespace token，需动态值时用 `RuntimeEnvService/readEnv()`；其他代码经 app-config，不在普通业务模块新增 process.env。DEMO_MODE、测试环境和动态身份设置不能缓存成永久快照。子进程 bootstrap、密钥等现有例外不能扩散成新例外。

## 新增变量的检查顺序

1. 在 app-config 派生和 schema 中定义类型、默认值、合法范围及读取时间。
2. 同步相应 token 或 runtime API、示例环境说明和测试，不在消费端各自 parse。
3. 布尔值按 boolean-like family 解析，不用字符串 truthiness；未知安全开关不应打开能力。
4. 端口/超时同时检查范围与运行时可表示性；不能仅 `> 0` 就接受小数或超大 timer。

测试证据：`server/tests/unit/app-config/validate.test.ts` 验证空配置/空白默认、多变量聚合错误以及 LLM_TIMEOUT_MS 小数、超 32 位范围的拒绝。不要把“未知变量忽略”误解成“已知变量任意值均允许”。

## 密钥与本地 HTTP

`server/src/config.ts` 区分 JWT 会话密钥与静态加密密钥。加密密钥来源按显式环境、专用文件、旧 JWT 文件兼容迁移、生成值处理；专用文件不可读或为空时拒绝启动，以免覆盖后丢失旧数据解密能力。JWT 密钥由服务管理，轮换后的 live binding 供验证器使用，不在新代码复制一份缓存。

`server/.env.example` 含演示 OIDC 与 HTTPS Cookie 配置，不能原样作为所有开发机要求。本地 HTTP 的 Cookie、APP_URL、origin 和首次登录说明见[本地开发](../../guides/local-development.md)。生产安全开关不能为了方便测试而全局关闭。

## 验证

根目录：`npm run test --workspace=server -- tests/unit/app-config tests/unit/nest/app-config.test.ts`，必要时增加 `tests/integration/app-config.test.ts` 的真实应用验证。检查环境变化后读值是否更新、关闭应用是否释放资源。不要以启动应用来“只查看配置”，启动会操作密钥和数据库。
