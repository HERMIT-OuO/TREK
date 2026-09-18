# i18n 与 HTML 净化

## 注册表、键和翻译

**当前实现**：`shared/src/i18n/languages.ts` 的 `SUPPORTED_LANGUAGES` 是语言来源，`SupportedLanguageCode` 从它推导；`getLocaleForLanguage`、`getIntlLanguage`、`isRtlLanguage` 处理显示与方向，`br` 映射为 `pt-BR`。不要复制语言清单或凭猜测改语言代码。

**新增代码约束**（`shared/CLAUDE.md`）：以 `en/` 为基准，为各语言维护相同领域文件和顶层键，保留 `{placeholder}` 的名称，给出真实本地化文案。键存在但值照抄英语不是完成翻译。调用者使用顶层点分键，不随意改为深层对象。

**检查强度必须说清**：

- `shared/scripts/i18n-parity.mjs` 的 `checkParity()` 检查文件集和键集；普通 `i18n:parity` 报告漂移但退出 0，`i18n:parity:strict` 才因漂移失败。
- `shared/src/i18n/i18n-parity.spec.ts` 的单测只强制文件集一致；键漂移只验证报告形状，不保证无缺键。
- `shared/src/i18n/i18n-placeholders.spec.ts` 检查**已有翻译**包含英文源的插值名，不检查不存在的译文，也不能证明译文质量。
- 添加语言还要检查消费者懒加载映射和测试 locale 映射，不能只改注册表即宣称全链路已支持。

## 转义与净化不可互换

`shared/src/sanitize/sanitize.ts` 是共同入口：

| 函数 | 适用边界 | 不能当作 |
| --- | --- | --- |
| `escapeHtml` | 把用户字符串插入 HTML 模板前转义五种元字符 | 完整 HTML 净化器、URL 白名单 |
| `sanitizeInlineHtml` | 翻译等行内 HTML；有限标签，显式 `ALLOWED_ATTR: []` 且关闭 data 属性 | 任意富文本/外链渲染器，或所有属性类别都被禁用的承诺 |
| `sanitizeRichTextHtml` | 确实需要段落、列表、链接的内容；有受限属性 | 允许任意 style、事件处理器、iframe 的通道 |

推荐文本优先；确有 HTML 模板时先转义插入值，再按输出场景净化。React 普通文本渲染无需人为变成 HTML。现存重复 escapeHtml 是兼容债务，不在新模块复制。

**Node/浏览器边界**：这里导入 `isomorphic-dompurify`，浏览器使用 DOMPurify，Node 侧由其 DOM 实现支持净化；不是 `window` 缺失时直接透传 HTML。`shared/vitest.config.ts` 未切到 jsdom 环境，`sanitize.spec.ts` 在默认 Node 环境测试这些函数。新增调用必须保留服务端净化能力，但这不代表整个客户端已经支持 SSR，也不能让 shared 反向依赖客户端 DOM/IndexedDB 模块。

白名单要按实际 DOMPurify 选项理解：源码未显式设置 `ALLOW_ARIA_ATTR: false`，不能仅凭 `ALLOWED_ATTR: []` 声称所有属性一概消失。净化测试不等于 URL 仅限 HTTPS、自动补齐外链 rel 或服务端 SSRF 防护；相应调用边界仍需单独处理。

**测试证据**：`shared/src/sanitize/sanitize.spec.ts` 验证 script、事件属性、style、SVG/MathML 与 javascript/data href 被去除，合法 https href 保留。净化能力不代表全项目完全没有富文本；源码中的“未来若有 Markdown”注释不能覆盖客户端实际 Markdown 用法。

## 验证

根目录：`npm run i18n:parity:strict --workspace=shared`，以及 `npm run test --workspace=shared -- src/i18n src/sanitize/sanitize.spec.ts`。不以普通 parity 退出 0 代替严格检查；新增渲染入口还需检查其真实调用链与端侧安全测试。
