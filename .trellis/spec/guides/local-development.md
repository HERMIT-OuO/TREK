# 本地开发与验证命令

## 首次运行：先准备隔离环境

以下是未来产品开发的命令，不是规范初始化任务已执行的结果。依据根及各包 `package.json`、`server/.env.example`、`client/vite.config.js`。

```bash
# 仓库根目录；安装会写 node_modules 并运行依赖安装脚本
npm ci
# 先按下述说明准备 server/.env，再运行；会构建、写开发数据并监听端口
npm run dev
```

Node 版本应满足当前依赖的 engines、仓库 CI/镜像约束与本地平台；不要把 SDK 的 `node >=18` 当成整仓最低版本保证。可使用自己的版本管理器，不要求某个本机路径或 IDE。

不要原样启用示例中的外部服务。首次本地 HTTP 开发可手动创建 `server/.env`，使用下列最小配置；若文件已存在，逐项合并，不能覆盖已有配置：

```dotenv
NODE_ENV=development
PORT=3001
FORCE_HTTPS=false
COOKIE_SECURE=false
APP_URL=http://localhost:5173
ALLOWED_ORIGINS=http://localhost:5173
OIDC_ONLY=false
DEMO_MODE=false
```

未配置身份提供者时移除示例 OIDC issuer/client/secret；不要使用示例账号、密钥或域名连接服务。dotenv 不默认覆盖进程中已有的环境变量，排障时也检查启动终端的变量。端口或访问入口变化后同步 APP_URL/origin 与代理。

Vite 默认 `5173`，代理 API、上传、WS、插件 frame、MCP 和选定 OAuth 路由到 `localhost:3001`；`/oauth/consent` 是 SPA 路由，不要给整个 `/oauth` 盲加代理。后端端口若变更，代理也需对应调整。

## 数据与首次登录

- **当前实现**：`server/src/db/database.ts` 的 `initDb()` 在模块导入时打开数据库并建表、迁移、种子初始化。默认文件为 `server/data/travel.db`；测试模式用 `:memory:`，显式 `TREK_DB_FILE` 可使用独立文件。
- `server/src/db/seeds.ts` 的 `seedAdminAccount()` 只在无用户时生效。没有同时配置管理员邮箱和密码时生成随机密码并在首次启动日志显示，登录后需要修改；正确配置 OIDC-only 时首次 SSO 流程另行处理。已有用户时补设环境变量不会重设密码。
- `server/src/config.ts` 自动持久化 `.jwt_secret` 与 `.encryption_key` 到 data 目录；加密密钥读取异常/空文件不能当作“缺失”随意重建，否则旧凭据不可解密。
- 启动、导入后端模块、E2E、备份恢复均可能改数据库或文件。不要用生产 data 做开发检查，不运行重置账号、清空数据库来修复普通测试失败。

## 构建、类型检查和调试不是同一件事

```bash
# 根目录；消费者使用 shared/dist，不直接使用 shared/src
npm run build --workspace=shared
npm run typecheck --workspace=shared
npm run typecheck --workspace=server
npm run typecheck --workspace=client
# SDK 独立安装；仅在开发 SDK 时需要
npm ci --prefix plugin-sdk
npm run typecheck --prefix plugin-sdk
```

**当前实现**：`server/scripts/build.mjs` 捕获 tsc 错误后仍输出 dist ready；`server/tsconfig.build.json` 设置 `noEmitOnError: false`。因此后端 build 成功不能视作类型检查通过，须单独运行 typecheck；测试类型另用 `typecheck:tests`。

`server/scripts/dev.mjs` 先构建，再启动 tsc watch，看到首次 watch 就绪信息才启动 `node --watch dist/index.js`。运行的是 JS；构建配置 `sourceMap: false`。默认 dev 没有开启 inspector，也不生成 TS 调试映射。

需要后端 TypeScript 断点时，可用下列独立终端流程，无需修改仓库配置。先准备上述隔离环境、在根目录构建 shared，并停止已有根/server dev，避免两个后端争用端口：

```bash
# 准备终端，仓库根目录；shared 变化后重新构建，或另开 build:watch
npm run build --workspace=shared

# 终端 A，初始 cwd 为仓库根；显式生成 .map 和内嵌源码，持续写 server/dist
cd server
npm exec -- tsc -p tsconfig.build.json --sourceMap true --inlineSources true --watch --preserveWatchOutput

# 终端 B，初始 cwd 为仓库根；等 A 首次编译完成后运行
cd server
node --inspect=127.0.0.1:9229 --enable-source-maps --require tsconfig-paths/register --watch dist/index.js

# 终端 C，仓库根目录；只启动前端，不能用根 dev 重启另一份后端
npm run dev --workspace=client
```

调试器 attach 到 `127.0.0.1:9229`，启用 Source Map，并把已编译文件位置指向 `server/dist/**/*.js`；重启后按调试器能力重新连接。排查启动阶段可用 `--inspect-brk=127.0.0.1:9229` 替换 `--inspect`。不要向公网开放 inspector。`--enable-source-maps` 只负责堆栈映射，不能凭空生成 `.map`，watch 发射也不等于 typecheck 通过。客户端生产 build 同样关闭 sourcemap，浏览器调试使用 Vite dev 的 DevTools。

## 检查的副作用

| 命令（根目录） | 覆盖/副作用 |
| --- | --- |
| `npm run test` | shared → server → client，非 SDK；测试可能写临时数据 |
| `npm run test:cov` | 三 workspace 加 SDK；产生 coverage |
| `npm run test:e2e` | **服务端 Vitest E2E**，不是浏览器 Playwright |
| `npm run e2e --workspace=client` | 浏览器测试，需 Playwright 前提，可能启动服务并写报告 |
| `npm run lint` | 根脚本会执行 shared/server 的 `--fix`，会改源码 |
| `npm run lint:check --workspace=server` / `--workspace=client` | 不自动修复 |
| `npm exec --workspace=shared -- eslint "src/**/*.ts"` | 已安装本地 ESLint 时的只检查替代命令；无 `--fix` |
| `npm run format` / `npm run format:check` | 前者改文件，后者只检查；按包脚本 glob，并非全仓所有文件 |
| `npm run build` | 构建三个 workspace；client prebuild 会生成图标；不构建 SDK |

包级定向测试和额外门禁见各包质量指南。不要把需要已有 dist 的 `check:gl-split` 当作源码检查，也不要为了简单文档改动安装依赖或全量构建。
