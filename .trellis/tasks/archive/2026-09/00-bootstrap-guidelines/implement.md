# 规范初始化执行计划

## 当前阶段

用户已在最终规划摘要后明确批准实施。任务已由 task.py start 激活为 in_progress；原始 36 份规范、config.yaml 和 4269 个已跟踪文件哈希已保存至本任务 `.baseline/`。下面按实际执行结果更新复选项。

## 0. 规划验收与激活

- [x] 用户明确批准最新 PRD、目录方案、范围与验收摘要。
- [x] 再次核对 git status、现有 spec 和其他任务上下文，确认规划期间无冲突改动。
- [x] `implement.jsonl`、`check.jsonl` 均有真实研究条目，`task.py validate` 通过。
- [x] 运行 `python3 ./.trellis/scripts/task.py start .trellis/tasks/00-bootstrap-guidelines` 成功，任务已激活；未手写 runtime 指针。

## 1. 保存基线并补齐证据

- [x] 将原有 spec 文件、config.yaml 的内容与哈希保存到任务的本地备份位置；记录任务外原有未提交改动，不依赖未跟踪文件的 Git 回退。
- [x] 完整读取拟删除的模板文件，确认没有需要保留的用户内容；20 份删除文件均与原始基线一致。
- [x] 按 `research/repository-analysis.md` 的补读清单逐主题读取源码与测试；续做证据及最终全范围复核见 research 下两份实施/审查记录。
- [x] 对重要规则记录真实路径和符号/测试名，区分当前实现、新增代码约束和兼容例外；全文审查修正了过强保证。

回退点 A：此阶段除任务研究/备份外无正式规范修改，可直接暂停。

## 2. 编写共享指南和包级规范

按以下顺序完成，避免同一规则重复散落：

- [x] guides：总体架构、跨层影响、复用原则、本地运行/调试。
- [x] shared/library：契约、类型导出、翻译和净化；统一前后端共同术语。
- [x] server/backend：模块、数据、接口权限、配置、错误日志、MCP/插件/存储等边界和质量检查。
- [x] client/frontend：组件/Hook/状态、离线与实时、主题地图、类型和测试。
- [x] plugin-sdk/sdk：API/CLI、发布包边界、协议镜像与生成文件、测试限制。
- [x] 更新所有 index.md：适用源码、开发前清单、质量检查入口和完整主题导航。

只用中文解释规则，不翻译代码标识符。示例应短小真实；新代码约束应有已存在约定或检查规则支撑。文档不能引导自动发布、提交或修改生产数据。

## 3. 迁移目录与包映射

- [x] 在内容已承接并检查引用后，移除 server/frontend、shared/frontend、shared/backend 中不适用的模板。
- [x] config.yaml 仅添加 plugin-sdk 包映射，将 default_package 改为 client；全文与预期基线替换一致。
- [x] 检查所有任务 JSONL 与规范相对链接：2 个真实引用均有效，没有其他进行中任务引用删除层。
- [x] 包发现输出显示 client/frontend、server/backend、shared/library、plugin-sdk/sdk，并标记 client 为默认包。

回退点 B：必要时按文件基线恢复本任务的规范/映射变更，仅删除本任务新建文件；恢复前核对当前哈希防止覆盖并行编辑。禁止全仓库 reset/clean。

## 4. 文档质量门禁

### 必跑的只读命令

```bash
python3 ./.trellis/scripts/get_context.py --mode packages
python3 ./.trellis/scripts/task.py validate .trellis/tasks/00-bootstrap-guidelines
python3 ./.trellis/scripts/task.py list-context .trellis/tasks/00-bootstrap-guidelines
# 以下搜索无匹配是预期结果，rg 此时返回 1；返回 2 才表示执行错误。
rg -n 'To be filled by the team|To fill|TODO: fill|Fill in each file|your-project|path/to/' .trellis/spec
# 只检查空白错误，不自动格式化。
git diff --check
git status --short
```

### 结构与证据检查

- [x] 扫描全部 31 份规范：无模板、空叶子标题段或孤立主题文件。
- [x] 83 个 Markdown 本地链接有效，标题片段通过检查；所有主题由同层 index 导航且从 guides 入口可达。
- [x] 主会话复核 171 个字面包内源码/配置路径；另在独立审查中核对根配置等引用，运行时路径与 glob 不误报为源码缺失。
- [x] 独立审查完整阅读 31 份规范并回查高风险源码与测试，修正 12 份文档中的不准确或过强表述。
- [x] 39 组包/脚本组合、29 个定向测试路径静态核对通过；未把命令存在当作本轮执行通过。
- [x] 跨文档核验 API 缓存、HTTP/Cookie、Node/浏览器净化、临时 ID、幂等、事件、鉴权、金额与 SDK 生成源等边界。
- [x] 两个 JSONL 各有真实研究 file/reason 条目，task.py validate 通过；无产品源码或被删除模板引用。
- [x] 额外检查未跟踪规范的空白与内容，git diff --check 通过。
- [x] 4269 个已跟踪文件哈希与基线一致，包含既有 .gitattributes 改动；工具写入限授权范围。
- [x] 记录诊断覆盖限制：主会话 lens 无阻塞发现但不覆盖全部 Markdown；配置 3 条既有超长注释保留，文档结构检查与独立审查均通过。

### 不默认执行的产品验证

本任务不改产品代码，不运行 npm ci、应用启动、全仓库 lint/test/build、生成器、发布或数据库操作。正式规范可以记载未来产品开发所需命令，但不能声称这些命令本轮通过。

## 5. 收尾与用户交付

- [x] 最终目录、引用校验、包发现、JSONL 和范围检查结果已记录在 verification.md 与 research/final-review.md。
- [x] PRD 的 AC1–AC6 均通过文档任务验收，无范围内阻塞问题。
- [x] 交付说明已整理，明确未运行产品验证和既有诊断限制。
- [x] 初次交付未自动提交/归档；后续按用户明确授权完成工作提交及任务归档，状态为 completed；未 push，详见 verification.md 收尾记录。
