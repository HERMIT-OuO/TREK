# 插件 SDK / CLI 开发规范

适用范围：仓库内独立包 `plugin-sdk/`，包名 `trek-plugin-sdk`。这是作者侧 API、测试宿主和 CLI，不是 TREK 的 npm 根 workspace，也不是插件生产隔离运行时。

## Pre-Development Checklist

- 阅读 `plugin-sdk/CLAUDE.md`，确认本次改公共 API、manifest、CLI、mock host 还是生成事实。
- 改权限、事件、调用方法或浏览器桥接前，找到宿主对应来源及 parity 测试。
- 区分 SDK 独立 checkout 可执行的检查与需要 TREK/registry 源码的检查。
- 不给 SDK 增加对 server/shared/client 源码的运行时相对导入，发布包中没有这些目录。
- 不手改生成文件；先读生成器输入、输出和只读 check 模式。
- 先确定 TTY/非 TTY、stdout/stderr 和退出码是否属于兼容面。
- 发布、打 tag、修改签名历史、访问远端 registry 不属于普通质量检查，需单独授权。

## 主题导航

| 主题 | 负责范围 |
| --- | --- |
| [公共 API 与 CLI](api-and-cli.md) | 导出、双格式构建、校验、交互和权限 |
| [宿主一致性](host-parity.md) | 生成事实、手工镜像、mock 保真与 skip 条件 |
| [质量检查](quality-guidelines.md) | 独立安装、定向测试、构建、检查副作用 |

## 当前包边界

- `plugin-sdk/package.json` 的两个 exports 为 `.` 与 `./testing`，各有 import/require 和声明文件入口。
- `plugin-sdk/src/index.ts` 定义作者 API；`definePlugin` 返回原对象，不是验证器或安全沙箱。
- `plugin-sdk/src/mock-host.ts` 提供权限约束与 fixture 驱动，不能替代真实数据库和隔离宿主集成测试。
- `plugin-sdk/src/cli/trek-plugin.ts` 是 `trek-plugin` / `trek-plugin-sdk` 共同路由；`create-trek-plugin` 另有 create 入口。
- `plugin-sdk/src/ui/kit.ts` 提供内联 CSS/JS 和消息桥助手；真正安全边界在宿主 iframe/进程/RPC。

## 新代码的方向

依据包级约定，宿主契约优先单一来源，不能以独立发布为理由放宽权限或另造协议。生成事实与纯函数镜像各有边界；新增镜像应提供能检测双向漂移的证据，而不是只验证“旧条目还在”。

SDK API 的开放字段是有意兼容，不要求与应用 REST 实体类型逐字段等同；修改时区分 API 版本、npm 包版本和 manifest 的 TREK 兼容范围。

## Quality Check

在 `plugin-sdk/` 使用 `npm run typecheck`、`npm test -- <测试文件>`；包没有 lint 脚本。公共导出变化再验证 `npm run build` 产生的 ESM/CJS 两套入口。

宿主协议改动还需 TREK 根目录的 `npm run check:plugin-facts --workspace=@trek/server` 及相应 parity 测试。独立 checkout 中 skipped 不能写成“宿主等价已通过”。完整说明见[质量检查](quality-guidelines.md)，服务端边界见[插件宿主规范](../../server/backend/integrations-and-plugins.md)。
