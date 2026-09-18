# 错误响应与失败可见性

## 当前响应封装

依据 `server/src/nest/common/trek-exception.filter.ts` 的 `TrekExceptionFilter.catch`：

| 错误来源 | 响应行为 |
| --- | --- |
| TREK 原生 HttpException 对象 `{ error, code? }`，无 Nest message/statusCode 字段 | 原样传递对象与状态 |
| Nest 常规 HttpException | 归一为 `{ error: string }`；常规 5xx 消息为 `Internal server error` |
| 普通 Error 携带 statusCode/status | 4xx 暴露 message，5xx 使用通用文本并记录错误 |
| 原始 MulterError | LIMIT_FILE_SIZE 为 413，其余为 400 |
| 已发送 headers 的流异常 | 记录并 destroy 响应，不再写第二份 JSON |

注意原生 `{ error }` 分支会原样传递，即使给了 5xx；因此不要将内部异常文本主动包装为可透出的 error。不能笼统声称 filter 自动净化所有 5xx 或所有日志。

## 在边界表达错误

- `ZodValidationPipe` 将字段 issue 拼成 `field: message`，根路径标记 body，返回 400。旧接口如 weather 的固定文案按 controller 与测试保留，不随手翻译或换成 Zod 默认结构。
- `CollectionsService` 目前通过带 status 的普通 Error 表达 404/403；这是 filter 支持的兼容方式。新增逻辑使用所属领域已有错误模型，不引入第二套通用 envelope。
- 权限错误必须保持资源可见性，见[API 与鉴权](api-and-auth.md)。不要将所有异常都改成 200 + success false，也不要 catch 后返回 null 掩盖外部服务失败。
- 服务操作与补偿失败要分别可观测；乐观 UI 的回滚/提示由调用链处理，不能让后端假成功掩盖部分写入。

## 流与外部 I/O

`server/src/nest/storage/storage.service.ts` 的 `sendToResponse` 区分本地 sendFile 与远端流，资源不存在由调用者按路由契约映射，客户端 abort 有专门处理。响应已经开始后再次 res.json 会损坏下载；不要在多个层重复写响应。

网络错误应保留必要的操作上下文但不泄露 URL 凭据、token 或 provider 完整响应。HTTP 与 MCP 的错误载体不同，业务权限/结果要一致，不要求把 MCP 工具错误伪装为 HTTP JSON。

## 验证

从根目录运行 `npm run test --workspace=server -- tests/unit/nest/exception-filter.test.ts` 并补所改接口的 4xx/5xx/流中断测试。对照真实状态、body、headers 是否发送，而不是只断言抛出了 Error。日志安全另见[日志指南](logging-guidelines.md)。
