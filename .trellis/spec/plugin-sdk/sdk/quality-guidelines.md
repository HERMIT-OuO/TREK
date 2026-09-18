# 插件 SDK 质量检查

适用范围：`plugin-sdk/` 的独立包开发。命令依据 `plugin-sdk/package.json`，不是本轮文档任务的执行记录。

## 安装与运行边界

- SDK 有自己的 `package-lock.json`，根 `npm ci` / workspace test 不会安装或验证它；根 `test:cov` 则显式用 `--prefix plugin-sdk` 追加 SDK coverage，不能误写为所有根脚本均排除 SDK。
- 产品开发需要安装时在 `plugin-sdk/` 运行 `npm ci`；不要为了省一步把 SDK 加入根 workspaces。
- package engines 声明 Node >=18；具体工具与 dev 的可选 node:sqlite 能力仍受所选运行时限制，不保证所有版本具备相同预览功能。
- 发布包不访问相邻 server/client/shared；仓库中的生成器和 parity 是维护流程，不是作者安装的前置依赖。

## 常规验证入口

以下均在 `plugin-sdk/`：

```bash
npm run typecheck
npm test -- test/sdk.test.ts test/cli.test.ts
npm test -- test/manifest-roundtrip.test.ts test/manifest-settings.test.ts
npm test -- test/permissions.test.ts test/mock-limits.test.ts
```

`typecheck` 不输出文件；没有 `lint` / `lint:check` 脚本，不虚构命令。`npm test` 运行全部 Vitest；测试可能创建临时目录、fixture、ZIP 或启动测试进程，不能描述为完全无副作用。

## 按变更选择证据

| 改动 | 至少核对 |
| --- | --- |
| definePlugin、ctx API | `plugin-sdk/test/sdk.test.ts`；允许、缺授权、非成员与无 acting user |
| manifest 字段 | roundtrip/settings 测试；归一化是否保留字段、默认 scope 与 secret 规则 |
| 权限/事件 | permissions 测试、permissions-parity、宿主事实 check、CLI picker 完整覆盖 |
| egress | `plugin-sdk/test/egress-policy.test.ts` 与 permissions-parity；拒绝/允许目标都测 |
| CLI 输出/交互 | cli 测试；非 TTY 不等待输入，stdout 数据不混提示，未知 flag 拒绝 |
| UI kit/桥 | `plugin-sdk/test/ui-kit.test.ts`、session parity；宿主与 dev 同样的 scope/错误码 |
| registry gate | `plugin-sdk/test/checks.test.ts`、checks-parity；区分 artifact 和 publish gate |

不能只让一个宽松 mock 通过就宣称宿主一致；parity 先读[一致性规范](host-parity.md)的 skip 条件。

## 构建和覆盖率会输出文件

- 公共 API/exports 变化时运行 `npm run build`，同时验证 ESM 与 CJS 输出和声明；不只检查 `dist/index.js`。
- 构建末尾 `scripts/finish-cjs.mjs` 生成 CJS 包作用域标记，不能省略。
- `npm run test:coverage` 写 coverage；`plugin-sdk/vitest.config.ts` 排除机器生成 host-facts 和 lucide 快照，未配置客户端同样的四项阈值，不能照搬客户端结论。
- 图标生成命令 `node scripts/gen-lucide-icon-names.mjs` 依赖根安装的 lucide-react，且改写源码快照；独立 checkout 不默认执行。

## 宿主与 registry 条件检查

在 TREK 根运行 `npm run check:plugin-facts --workspace=@trek/server`，只检查漂移；不以无参数生成器替代只读检查。

在 SDK 目录运行 `npm test -- test/permissions-parity.test.ts test/checks-parity.test.ts`。记录是否真的执行到断言：独立 SDK 缺宿主会 skip；registry 测试还需独立仓库，可由 `TREK_PLUGINS_REPO` 指定。registry 的离线 parity 不代替联网 preflight。

## 不能当作检查的操作

`create`、`dev`、`pack`、`shot` 会创建文件、执行插件或启动服务；发布、签名、key rotation、release/tag/submit/unrelease 还涉及私钥、远端或不可逆历史。只有任务明确需要且得到授权时才操作。

`prepublishOnly` 会 build + test，但不是推荐的检查入口，更不能用 npm publish 验证代码。正常 status 是方向提示，不是 CI gate；validate 仅保证执行到的离线 gate。

完成后报告路径、测试结果、产物和 skip/未执行原因。纯文档任务无需安装、运行产品测试、生成源码或构建发布包；源码阅读不等于产品验证通过。
