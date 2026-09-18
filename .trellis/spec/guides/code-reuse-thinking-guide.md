# 复用前先确认数据所有者

## 修改前的检查顺序

1. 找符号定义、调用者、相邻测试，再决定新增还是复用；修改常量也要检查生成源和消费端。
2. 识别重复的是同一业务规则，还是恰好相似的显示。不要因为两段代码长得像，就创建全局抽象。
3. 抽取到拥有语义的最小层：页面纯计算放 Model，桌面/手机共享行为放 Hook，线协议放 shared，数据库操作留后端。

这些原则承接 `client/CLAUDE.md`、`shared/CLAUDE.md` 的新增代码约束，不要求为每个重复的一行表达式建工具库。

## TREK 中应优先复用的入口

| 问题 | 先检查的证据 | 避免 |
| --- | --- | --- |
| 新增 URL ID 校验 | `shared/src/common/primitives.schema.ts`：`idParamSchema` 与 `idSchema` | 各 controller 自行 parseInt 后漏掉负数/非整数 |
| 日期中的 AM/PM | `shared/src/datetime/datetime-normalize.ts`：`splitLocalDateTime`、`normalizeLocalDateTime` | 直接 slice 丢掉半天，或把当地时间误转 UTC |
| 金额类别 | `shared/src/budget/budget.schema.ts`：`COST_CATEGORIES`、`typeToCostCategory` | 前后端各写一套预订类型映射 |
| HTML 字符串 | `shared/src/sanitize/sanitize.ts`：`escapeHtml`、两种净化器 | 在新组件再复制 escapeHtml，或拿转义替代 URL 校验 |
| 手机和桌面相同行为 | `client/src/pages/PATTERN.md` 的 Hook/Model 分工；[Hook 指南](../client/frontend/hook-guidelines.md) | 在两个壳重复请求、权限判断和错误处理 |
| 插件权限常量 | `shared/src/index.ts` 对生成权限的导出；[生成源说明](../plugin-sdk/sdk/host-parity.md) | 手改生成结果，或让 mock 自己发明一套权限 |

## 不该机械统一的地方

**历史兼容**：weather 经纬度保持字符串、旧 trip 布尔字段混合 0/1 与 boolean，是既有线协议。不要为了“统一类型”只改 Schema；应先设计兼容迁移并验证真实请求/响应。

**当前实现**：Shared 负责共同语义，但 SDK 是独立发布包；宿主镜像有专门 parity 测试，不能用跨目录源码 import 破坏独立包。

## Quality Check

抽取后对照行为而不只看行数：桌面和手机是否仍发同样请求、返回同样错误？所有调用者是否改用统一入口？测试是否能在删除该行为后失败？生成路径是否仍可重现？具体命令进入对应包质量指南。
