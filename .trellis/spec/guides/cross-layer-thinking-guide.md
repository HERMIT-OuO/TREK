# 跨层变更检查

## 先画完整路径

涉及契约、离线、权限、事件、插件的改动，即使只改一个文件，也先列出：

```text
输入 → Schema/身份 → 领域服务 → 存储 → 响应 → Repository/缓存 → Store → 桌面与手机
                                  └→ 实时事件 / MCP / 插件能力
```

每个箭头确认字段名、ID 类型、金额单位、日期语义、null/缺省、错误及数据所有者。不要将“边界统一校验”误写成“全链路只校验一次”：外部 HTTP、MCP、插件 RPC 是不同信任边界，均需验证，内部消费者复用统一契约。

## 重点契约

| 变更 | 必须联查 | 证据与常见失误 |
| --- | --- | --- |
| Schema 字段 | 请求、响应、导出、两端消费者及存量数据 | `shared/src/collection/collection.schema.ts` 有输入 URL 收紧、输出保留旧值的区别；不要用一个严 Schema 直接吞掉旧数据 |
| 写入与重试 | 临时负 ID、UUID 幂等键、事务、冲突、回滚 | [离线同步](../client/frontend/offline-sync.md)；不要让重试重新生成一个创建操作 |
| 实时变更 | 事件注册表、REST/MCP 发射、客户端缓存和 reducer | `shared/src/realtime/events.schema.ts` 的 `TREK_WS_EVENTS`；HTTP 成功但其他设备无变化也是缺陷 |
| 权限变化 | 默认鉴权、领域权限、信息可见性、MCP、离线残留 | [API 与鉴权](../server/backend/api-and-auth.md)；401/403/404 不能随手互换 |
| 金额/时间 | 字段币种、换算点、当地时间与时间戳 | `budgetItemPayerSchema.amount` 是费用自己的币种；`normalizeLocalDateTime` 保留本地日期时间语义；不能假定所有数字都是分或所有日期都是 UTC |
| 语言 | 注册表、文件/键、插值、懒加载导出 | `shared/src/i18n/languages.ts`；键齐全不等于已翻译或插值正确 |
| 插件 API | 宿主实现、协议、SDK、mock、生成事实、权限测试 | [host parity](../plugin-sdk/sdk/host-parity.md)；独立 SDK checkout 的 skip 不等于真实宿主验证通过 |

## 验证问题清单

- 正常、缺字段、空值、错误类型、旧数据能否完成往返？Schema parse 有没有丢掉尚未声明的新字段？
- 乐观 UI、离线回放、在线 REST、MCP、远端 WS 最终是否得到一致实体？哪些领域明确在线专用？
- 同设备换用户、服务不可达、403、409、重放、断线重连是否有明确行为？
- 外部 URL 是否既做格式验证，又在服务端出站请求处做 SSRF 防护？HTML 净化不能代替网络策略。
- 判断审查问题时追踪真实输入来源、调用链和测试，不以文件名或注释断言存在漏洞，也不以“内部数据”跳过实际不可信输入。

命令按[本地开发](local-development.md)与包级质量指南选择。先验证改动触及的边界，再决定是否需要全量验证；记录失败与未执行项，不用 mock 成功冒充跨端集成成功。
