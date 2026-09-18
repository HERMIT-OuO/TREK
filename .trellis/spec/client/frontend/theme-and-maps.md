# 主题与地图

适用范围：宿主 UI 外观、手机壳和地图渲染/缓存。依据 `client/src/theme/README.md`、当前主题代码及地图构建入口。

## 新增代码约束：使用已定义语义 token

`client/tailwind.config.js` 将语义类映射到 CSS 变量，`client/src/index.css` 提供实际方案值。

| 用途 | 使用方式 |
| --- | --- |
| 背景/文字/边框 | `bg-surface*`、`text-content*`、`border-edge*` |
| 主操作 | `bg-accent text-accent-text`；表面强调文字用 `text-accent-on` |
| 状态 | `bg-danger-soft text-danger` 等 success/danger/warning/info 组合 |
| 字号层级 | `text-title/subtitle/body/caption` |
| 浮层 | 已有 shadow token 与 `--z-modal`、`--z-toast` 等层级 |

不新增 palette 类、硬编码 hex/rgba、任意颜色类或虚构 CSS 变量。动态数据颜色、几何尺寸、地图 paint 等无法直接消费 token 的边界允许有意例外，按主题文档标注 `theme-lint-disable`，不要整文件豁免普通 UI。

## 当前实现：外观写入和首屏

`client/src/theme/applyAppearance.ts` 的 `applyAppearance` 归一化 shared `AppearanceConfig`，更新 html 的 dark、scheme、透明度、密度、减弱动画及自定义颜色/字号变量。

- 组件读取 token，不读取配置重复计算自己的配色。
- 运行期外观集中在这一入口；`client/public/theme-boot.js` 是首屏前回放缓存快照的特例，修改快照必须同步两端。
- shared/public 页面强制浅色、中性配色和透明度；不要进一步声称所有用户字号/密度偏好都清空。
- `clearAppearanceSnapshot` 用于退出清理，避免下一账号闪现上一账号外观。
- `client/src/theme/applyAppearance.test.ts` 验证默认无标记、布尔/字符串 dark、auto、共享页中性外观和自定义 accent。

## 当前实现：手机 token 有作用域

`client/tailwind.config.js` 的 `m-*` 对应 `client/src/mobile/mobile.css`，只在 `.m-root` 内有意义。不要发明第二套全局 token，也不要把现存手机 token 当成不存在。

`client/src/components/Plugins/PluginFrame.tsx` 的 `readThemeTokens` 分别在 documentElement 和 `.m-root` 读取全局/手机变量，通过消息桥给插件；`--glass-*` 等局部层不能在根节点读取后假装有效。

## 当前实现：两类地图、两种 GL 引擎

- `client/src/components/Map/MapViewAuto.tsx` 按设置选 Leaflet 或 GL；Mapbox 无 token 时回退 Leaflet。GL 加载/失败也以 Leaflet 为 fallback。
- `client/src/components/Map/glLazy.tsx` 分别动态导入组件与 `engines/mapbox`、`engines/maplibre`，并行启动，engine 作为 prop 注入共享组件。
- 新地图能力检查 Leaflet 与 GL 表现；不能在共享 core 静态导入两个引擎，从而合回大 chunk。
- Atlas 的地图直接使用 Leaflet，不能推断设置切换会替换所有地图。

## 当前实现：预下载不能只看旧注释

`client/src/sync/glPrefetcher.ts` 已实现矢量 style、TileJSON、sprite、glyph 和 tile 预下载，并直接写 `gl-map-offline`。`client/src/sync/tilePrefetcher.ts` 的 `prefetchTilesForTrip` 根据解析后的 tile 模板是否为 vector style 分流，不是按 GL renderer 名称统一开启。“所有 GL 都只机会缓存、完全没有预下载”是旧注释，不是当前统一事实。

`client/vite.config.js` 的规则有顺序：Mapbox/OpenFreeMap 的 **style 文档先命中 NetworkFirst 的 `gl-map-styles`**；OpenFreeMap 其余资源才命中 CacheFirst 的 `gl-map-offline`。预取器写入后者不等于 style 文档请求一定读取后者，不能概括为所有 GL 资源共享一个缓存。验证离线地图时同时检查 style 的服务缓存和底图资源，不能只数已预取 tile。

Mapbox 资源仍有机会缓存规则，不能把 OpenFreeMap 预下载能力推广给任意供应商。栅格预下载使用 no-cors；矢量需要读取响应，使用 cors，不能为了形式一致互换。

API 不走地图缓存。扩展供应商时核对缓存匹配、容量、资源 URL 和 CORS；业务实体离线另见[离线同步](offline-sync.md)。

## 验证与扫描限制

在 `client/` 运行 `npm run theme:lint` 和 `npm test -- src/theme/applyAppearance.test.ts`。默认扫描只报告且返回 0；`theme:lint:strict` 才因残留失败，且命名 palette 类要人工检查。

GL 改动需先 `npm run build` 再 `npm run check:gl-split`，脚本按两个特征字符串检查 `dist/assets`，同 chunk 命中两者或完全找不到引擎标记时失败；它不强制两种引擎都存在。没有构建产物不能当作检查通过。构建会生成文件，纯文档任务不执行。界面还要分别检查两个引擎、明暗、自定义 accent、手机作用域和失败 fallback。
